```bash
➜ anchor git:(main) ✗ export QN_HTTP="https://sparkling-intensive-silence.solana-devnet.quiknode.pro/d0132feb23407dcc8f9b73fdc967a35c591776e9/"

➜ anchor git:(main) ✗ export QN_WSS="wss://sparkling-intensive-silence.solana-devnet.quiknode.pro/d0132feb23407dcc8f9b73fdc967a35c591776e9/"
➜ anchor git:(main) ✗ export KEYPAIR="$HOME/my-keypair.json"
➜  anchor git:(main) ✗ solana config set --url "$QN_HTTP"
Config File: /Users/dylan/.config/solana/cli/config.yml
RPC URL: https://sparkling-intensive-silence.solana-devnet.quiknode.pro/d0132feb23407dcc8f9b73fdc967a35c591776e9/
WebSocket URL: wss://sparkling-intensive-silence.solana-devnet.quiknode.pro/d0132feb23407dcc8f9b73fdc967a35c591776e9/ (computed)
Keypair Path: /Users/dylan/my-keypair.json
Commitment: confirmed
➜ anchor git:(main) ✗ solana config set --keypair "$KEYPAIR"
Config File: /Users/dylan/.config/solana/cli/config.yml
RPC URL: https://sparkling-intensive-silence.solana-devnet.quiknode.pro/d0132feb23407dcc8f9b73fdc967a35c591776e9/
WebSocket URL: wss://sparkling-intensive-silence.solana-devnet.quiknode.pro/d0132feb23407dcc8f9b73fdc967a35c591776e9/ (computed)
Keypair Path: /Users/dylan/my-keypair.json
Commitment: confirmed
➜  anchor git:(main) ✗ solana cluster-version
2.2.12
➜  anchor git:(main) ✗ solana balance
1.96651776 SOL
➜  anchor git:(main) ✗ export SOLANA_DISABLE_TPU=1
➜  anchor git:(main) ✗
solana program deploy \
  --program-id target/deploy/counter_anchor-keypair.json \
  target/deploy/counter_anchor.so \
  --url "$QN_HTTP" \
 --output json
{
"programId": "DEWptrf6sj7CGL62SWuSCDvkdTZoyHQzXk3zknRgUdWS",
"signature": "wAUDJJoPT1JYFpzJYmsKgusEXJTjBNTaPSyPoH7d1PyBMfB8JFDAJhWbt4bvtVqmdTG7ZY5cuRJ4YeYn9qgRfm3"
}
```
