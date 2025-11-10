# Codex Passport 🛂

A cryptographic access gateway for Codexphere's LLM services using Ed25519 signature verification.

## Architecture Overview

Codex Passport is a secure authentication system that grants access to LLM services only after server-side verification of Ed25519 cryptographic signatures. The system implements a "passport" concept where users prove ownership of their public key through digital signatures.

### System Flow

```
User Input → Signature Verification → Find/Create Passport → Check Usage → LLM Proxy → Streaming Response
```

## Tech Stack

- **Frontend**: Next.js 14+ (App Router), React, TypeScript
- **Styling**: Tailwind CSS with Glassmorphism effects
- **Database**: Supabase (PostgreSQL)
- **Authentication**: Ed25519 cryptographic signatures via @noble/ed25519
- **LLM Integration**: Vercel AI SDK for streaming responses
- **Deployment**: Vercel (Edge Runtime)
- **Animations**: Lottie, Framer Motion
- **Syntax Highlighting**: Shiki or Prism

## Security Architecture

### Ed25519 Signature Verification

1. **Client-Side Signing**: User signs a challenge/message with their private key
2. **Server-Side Verification**: API route verifies signature using public key
3. **Zero Trust**: Private keys never leave the client, server only sees signatures

### Security Measures

- Rate limiting per IP and per passport (Upstash Redis)
- CORS protection with strict origin policies
- Content Security Policy headers
- Input sanitization and validation
- SQL injection prevention via parameterized queries
- CSRF protection via SameSite cookies

## Database Schema

```sql
-- Passports table
CREATE TABLE passports (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  public_key TEXT UNIQUE NOT NULL,
  passport_id TEXT UNIQUE NOT NULL, -- Format: cdx-{first8chars}
  tier TEXT DEFAULT 'free' CHECK (tier IN ('free', 'pro')),
  usage_count INTEGER DEFAULT 0,
  usage_limit INTEGER DEFAULT 100,
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'limit_reached')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_used_at TIMESTAMPTZ,
  metadata JSONB DEFAULT '{}'::jsonb
);

-- Usage logs table
CREATE TABLE usage_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  passport_id UUID REFERENCES passports(id),
  request_type TEXT,
  tokens_used INTEGER,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  ip_address INET,
  user_agent TEXT
);

-- Create indexes
CREATE INDEX idx_passports_public_key ON passports(public_key);
CREATE INDEX idx_passports_status ON passports(status);
CREATE INDEX idx_usage_logs_passport_id ON usage_logs(passport_id);
CREATE INDEX idx_usage_logs_timestamp ON usage_logs(timestamp);
```

### Supabase RPC Function

```sql
CREATE OR REPLACE FUNCTION find_or_create_passport(
  p_public_key TEXT,
  p_ip_address INET,
  p_user_agent TEXT
)
RETURNS TABLE (
  passport_id TEXT,
  tier TEXT,
  usage_count INTEGER,
  usage_limit INTEGER,
  status TEXT,
  can_proceed BOOLEAN
) AS $$
DECLARE
  v_passport_id TEXT;
  v_tier TEXT;
  v_usage_count INTEGER;
  v_usage_limit INTEGER;
  v_status TEXT;
  v_can_proceed BOOLEAN;
  v_db_passport_id UUID;
BEGIN
  -- Find or create passport
  INSERT INTO passports (public_key, passport_id, tier, usage_limit)
  VALUES (
    p_public_key,
    'cdx-' || substring(p_public_key, 1, 8),
    'free',
    100
  )
  ON CONFLICT (public_key) DO UPDATE
  SET last_used_at = NOW()
  RETURNING 
    passports.id,
    passports.passport_id,
    passports.tier,
    passports.usage_count,
    passports.usage_limit,
    passports.status
  INTO v_db_passport_id, v_passport_id, v_tier, v_usage_count, v_usage_limit, v_status;

  -- Check if can proceed
  v_can_proceed := v_status = 'active' AND v_usage_count < v_usage_limit;

  IF v_can_proceed THEN
    -- Increment usage count
    UPDATE passports SET usage_count = usage_count + 1 WHERE id = v_db_passport_id;
    v_usage_count := v_usage_count + 1;
    
    -- Log usage
    INSERT INTO usage_logs (passport_id, request_type, ip_address, user_agent)
    VALUES (v_db_passport_id, 'llm_request', p_ip_address, p_user_agent);
    
    -- Update status if limit reached
    IF v_usage_count >= v_usage_limit THEN
      UPDATE passports SET status = 'limit_reached' WHERE id = v_db_passport_id;
      v_status := 'limit_reached';
    END IF;
  END IF;

  RETURN QUERY SELECT v_passport_id, v_tier, v_usage_count, v_usage_limit, v_status, v_can_proceed;
END;
$$ LANGUAGE plpgsql;
```

## API Routes

### POST /api/llm-reply

Protected endpoint for LLM requests.

**Request Body:**
```json
{
  "codeSnippet": "console.log('Hello World');",
  "passportSignature": "hex_encoded_signature",
  "passportPublicKey": "hex_encoded_public_key"
}
```

**Response (Streaming):**
```
data: {"type":"passport","data":{"passportId":"cdx-1a2b3c4d","tier":"free","usage":"85/100","status":"active"}}

data: {"type":"chunk","data":"Here's an analysis..."}

data: {"type":"done"}
```

## Environment Variables

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# OpenAI
OPENAI_API_KEY=your_openai_key

# Rate Limiting (Upstash Redis)
UPSTASH_REDIS_REST_URL=your_redis_url
UPSTASH_REDIS_REST_TOKEN=your_redis_token

# Security
ALLOWED_ORIGINS=https://codexphere.com,http://localhost:3000
```

## Setup Instructions

### 1. Clone and Install

```bash
git clone https://github.com/yourusername/codex-passport.git
cd codex-passport
npm install
```

### 2. Database Setup

1. Create a Supabase project
2. Run the SQL schema from above
3. Create the RPC function
4. Copy environment variables to `.env.local`

### 3. Development

```bash
npm run dev
```

Visit `http://localhost:3000`

### 4. Deployment

```bash
# Deploy to Vercel
vercel --prod
```

## Client-Side Usage

```typescript
import { ed25519 } from '@noble/ed25519';

// Generate keypair (one time)
const privateKey = ed25519.utils.randomPrivateKey();
const publicKey = await ed25519.getPublicKey(privateKey);

// Sign a message
const message = new TextEncoder().encode('access-request');
const signature = await ed25519.sign(message, privateKey);

// Make request
const response = await fetch('/api/llm-reply', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    codeSnippet: 'your code here',
    passportSignature: Buffer.from(signature).toString('hex'),
    passportPublicKey: Buffer.from(publicKey).toString('hex')
  })
});
```

## Rate Limiting

- **Per IP**: 50 requests per hour
- **Per Passport**: Enforced via usage limits in database
- **Global**: 1000 requests per minute

## Features

✅ Ed25519 cryptographic authentication  
✅ Streaming LLM responses  
✅ Usage tracking and limits  
✅ Glassmorphism UI design  
✅ Animated backgrounds (falling stars)  
✅ Lottie animations  
✅ Syntax highlighting  
✅ Rate limiting  
✅ Comprehensive error handling  
✅ Mobile responsive  
✅ Dark cosmic theme  

## License

MIT

## Contributing

Contributions welcome! Please read our contributing guidelines first.

## Support

For issues or questions, please open a GitHub issue or join our Discord.
