# AiSolanaTokenGates# Solana Token Gate Implementation Guide

This guide explains how to implement token gating functionality in a React application using Solana Web3.js and wallet adapters.

## Overview

Token gating restricts access to certain features based on whether users hold a specific token and/or a minimum balance. This implementation uses:

- Solana Web3.js for blockchain interactions
- Wallet Adapter for connecting Solana wallets
- React hooks for state management
- TypeScript for type safety

## Installation

```bash
npm install @solana/web3.js @solana/wallet-adapter-react @solana/wallet-adapter-react-ui @solana/wallet-adapter-base @solana/wallet-adapter-wallets
```

## Implementation Steps

### 1. Create Token Gate Hook

Create `useTokenGate.ts`:

```typescript
import { useState, useEffect } from 'react';
import { useWallet } from '@solana/wallet-adapter-react';
import { Connection, PublicKey } from '@solana/web3.js';

export const useTokenGate = () => {
  const { publicKey } = useWallet();
  const [hasAccess, setHasAccess] = useState(false);
  const [tokenBalance, setTokenBalance] = useState(0);
  const [loading, setLoading] = useState(true);

  // Replace with your token's mint address
  const TOKEN_MINT = new PublicKey('YOUR_TOKEN_MINT_ADDRESS');
  const MIN_TOKENS_REQUIRED = 1;

  useEffect(() => {
    const checkTokenBalance = async () => {
      if (!publicKey) {
        setHasAccess(false);
        setLoading(false);
        return;
      }

      try {
        setLoading(true);
        const connection = new Connection('YOUR_RPC_URL');
        
        const accounts = await connection.getParsedTokenAccountsByOwner(
          publicKey,
          { mint: TOKEN_MINT }
        );

        const balance = accounts.value.reduce((total, account) => {
          return total + account.account.data.parsed.info.tokenAmount.uiAmount;
        }, 0);

        setTokenBalance(balance);
        setHasAccess(balance >= MIN_TOKENS_REQUIRED);
      } catch (error) {
        console.error('Error checking token balance:', error);
        setHasAccess(false);
      } finally {
        setLoading(false);
      }
    };

    checkTokenBalance();
    const interval = setInterval(checkTokenBalance, 30000); // Poll every 30 seconds
    return () => clearInterval(interval);
  }, [publicKey]);

  return { hasAccess, tokenBalance, loading };
};
```

### 2. Create Token Gate Component

Create `TokenGate.tsx`:

```typescript
import React from 'react';
import { useTokenGate } from '../hooks/useTokenGate';

interface TokenGateProps {
  children: React.ReactNode;
}

const TokenGate: React.FC<TokenGateProps> = ({ children }) => {
  const { hasAccess, tokenBalance, loading } = useTokenGate();

  if (loading) {
    return (
      <div className="flex items-center justify-center min-h-[200px]">
        <div className="animate-spin rounded-full h-8 w-8 border-t-2 border-b-2 border-purple-500"></div>
      </div>
    );
  }

  if (!hasAccess) {
    return (
      <div className="bg-gray-900 p-8 rounded-lg text-center">
        <h2 className="text-2xl font-bold text-white mb-4">Access Required</h2>
        <p className="text-gray-400 mb-6">
          You need at least 1 token to access this feature.
          {tokenBalance > 0 && (
            <span className="block mt-2">
              Current balance: {tokenBalance}
            </span>
          )}
        </p>
        <a
          href="YOUR_TOKEN_PURCHASE_URL"
          target="_blank"
          rel="noopener noreferrer"
          className="inline-block px-6 py-3 bg-purple-600 rounded-lg text-white"
        >
          Get Tokens
        </a>
      </div>
    );
  }

  return <>{children}</>;
};

export default TokenGate;
```

### 3. Setup Wallet Provider

In your `App.tsx` or main component:

```typescript
import { WalletProvider } from './contexts/WalletContext';

function App() {
  return (
    <WalletProvider>
      <YourApp />
    </WalletProvider>
  );
}
```

Create `WalletContext.tsx`:

```typescript
import { FC, useMemo } from 'react';
import { ConnectionProvider, WalletProvider } from '@solana/wallet-adapter-react';
import { WalletModalProvider } from '@solana/wallet-adapter-react-ui';
import { PhantomWalletAdapter } from '@solana/wallet-adapter-wallets';
import { clusterApiUrl } from '@solana/web3.js';

require('@solana/wallet-adapter-react-ui/styles.css');

export const WalletContext: FC<{ children: React.ReactNode }> = ({ children }) => {
  const endpoint = useMemo(() => clusterApiUrl('mainnet-beta'), []);
  const wallets = useMemo(() => [new PhantomWalletAdapter()], []);

  return (
    <ConnectionProvider endpoint={endpoint}>
      <WalletProvider wallets={wallets} autoConnect>
        <WalletModalProvider>{children}</WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
};
```

### 4. Usage

Wrap any component or feature you want to token gate:

```typescript
import TokenGate from './components/TokenGate';

function ProtectedFeature() {
  return (
    <TokenGate>
      {/* Your protected content here */}
      <div>This content is only visible to token holders</div>
    </TokenGate>
  );
}
```

## Configuration

1. Replace `YOUR_TOKEN_MINT_ADDRESS` with your token's mint address
2. Replace `YOUR_RPC_URL` with your Solana RPC endpoint
3. Replace `YOUR_TOKEN_PURCHASE_URL` with the URL where users can acquire tokens
4. Adjust `MIN_TOKENS_REQUIRED` to set the minimum balance requirement

## Best Practices

1. **Rate Limiting**: Implement rate limiting for RPC calls to avoid hitting API limits
2. **Error Handling**: Add comprehensive error handling for network issues
3. **Caching**: Cache token balances to reduce RPC calls
4. **Loading States**: Show appropriate loading states during checks
5. **Fallback Content**: Provide meaningful feedback when access is denied

## Security Considerations

1. **Client-Side Verification**: Remember that client-side checks are not sufficient for sensitive operations
2. **Backend Validation**: Implement server-side verification for critical features
3. **RPC Endpoint Security**: Use private RPC endpoints for production
4. **Balance Updates**: Monitor balance changes through WebSocket connections when possible

## Example Rate Limiter Implementation

```typescript
class RateLimiter {
  private queue: Array<() => Promise<any>> = [];
  private processing = false;
  private lastRequestTime = 0;
  private readonly minDelay: number;

  constructor(requestsPerSecond: number) {
    this.minDelay = 1000 / requestsPerSecond;
  }

  async schedule<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try {
          const result = await fn();
          resolve(result);
        } catch (error) {
          reject(error);
        }
      });

      if (!this.processing) {
        this.processQueue();
      }
    });
  }

  private async processQueue() {
    if (this.queue.length === 0) {
      this.processing = false;
      return;
    }

    this.processing = true;
    const now = Date.now();
    const delay = Math.max(0, this.minDelay - (now - this.lastRequestTime));

    if (delay > 0) {
      await new Promise(resolve => setTimeout(resolve, delay));
    }

    const fn = this.queue.shift();
    if (fn) {
      this.lastRequestTime = Date.now();
      await fn();
    }

    await this.processQueue();
  }
}
```

## License

MIT License - See LICENSE file for details
