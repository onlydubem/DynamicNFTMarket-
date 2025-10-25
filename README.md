# DynamicNFTMarket Smart Contract

A comprehensive NFT marketplace supporting multiple sale types, creator royalties, and advanced trading features on the Stacks blockchain.

## Overview

DynamicNFTMarket is a full-featured NFT marketplace that supports fixed-price listings, time-based auctions, and buyer-initiated offers. The contract includes automatic royalty distribution to creators and platform fee collection.

## Features

- **Multiple Sale Types**: Fixed-price sales, timed auctions, and offers
- **Creator Royalties**: Automatic royalty payments on every sale
- **English Auctions**: Time-based bidding with reserve prices
- **Offer System**: Buyers can make offers on any NFT
- **Sales History**: Complete on-chain transaction history
- **Collection Royalties**: Default royalties at collection level
- **Flexible Fee Structure**: Configurable platform and royalty fees
- **Automatic Refunds**: Previous bidders automatically refunded

## Sale Types

### Fixed-Price Listings
- Instant purchase at set price
- Seller can unlist anytime if no offers pending
- Automatic fee distribution on sale

### Auctions
- Time-based English auctions
- Reserve price protection
- Minimum bid increments (5%)
- Automatic refunds to outbid users
- Finalization after auction ends

### Offers
- Buyers lock funds when making offers
- Time-limited expiration
- Sellers can accept at any time
- Automatic cancellation and refund

## Key Functions

### Listing Management

#### `list-nft-fixed`
```clarity
(list-nft-fixed (nft-contract principal) (token-id uint) (price uint) 
                (royalty-recipient (optional principal)) (royalty-bps uint))
```
List NFT for fixed-price sale.

**Parameters:**
- `nft-contract`: NFT contract address
- `token-id`: Token ID
- `price`: Sale price in microSTX
- `royalty-recipient`: Optional royalty recipient address
- `royalty-bps`: Royalty percentage in basis points (0-5000 = 0-50%)

#### `list-nft-auction`
```clarity
(list-nft-auction (nft-contract principal) (token-id uint) (min-bid uint) 
                  (reserve-price uint) (duration uint)
                  (royalty-recipient (optional principal)) (royalty-bps uint))
```
List NFT for auction.

**Parameters:**
- `min-bid`: Starting bid amount
- `reserve-price`: Minimum acceptable price
- `duration`: Auction duration in blocks (minimum ~1 day)

#### `unlist-nft`
```clarity
(unlist-nft (nft-contract principal) (token-id uint))
```
Remove NFT from sale (only if no active bids).

### Purchasing

#### `buy-nft`
```clarity
(buy-nft (nft-contract principal) (token-id uint))
```
Purchase NFT at fixed price. Automatically distributes platform fees and royalties.

#### `place-bid`
```clarity
(place-bid (nft-contract principal) (token-id uint) (bid-amount uint))
```
Place bid on auction. Previous bidder automatically refunded.

**Requirements:**
- Auction must be active
- Bid must exceed current bid by 5%
- Bid must meet minimum or reserve price

#### `finalize-auction`
```clarity
(finalize-auction (nft-contract principal) (token-id uint))
```
Complete auction after end time if reserve met. Distributes funds to all parties.

### Offer System

#### `make-offer`
```clarity
(make-offer (nft-contract principal) (token-id uint) (amount uint) (expiration uint))
```
Create offer with locked funds.

#### `cancel-offer`
```clarity
(cancel-offer (nft-contract principal) (token-id uint))
```
Cancel offer and receive refund.

#### `accept-offer`
```clarity
(accept-offer (nft-contract principal) (token-id uint) (offerer principal))
```
NFT owner accepts offer. Funds distributed automatically.

### Read-Only Functions

- `get-listing`: Retrieve listing details
- `get-auction`: Retrieve auction information
- `get-offer`: Get specific offer details
- `get-royalty-info`: Check royalty configuration
- `calculate-fees`: Calculate distribution for sale price

### Admin Functions

#### `set-platform-fee`
```clarity
(set-platform-fee (new-fee uint))
```
Update platform fee (max 10%, in basis points).

#### `set-min-auction-duration`
```clarity
(set-min-auction-duration (new-duration uint))
```
Set minimum auction length in blocks.

#### `register-collection-royalty`
```clarity
(register-collection-royalty (nft-contract principal) (recipient principal) (percentage uint))
```
Set default royalties for entire NFT collection.

## Usage Examples

### Fixed-Price Sale

```clarity
;; 1. List NFT for 500 STX with 10% royalty
(contract-call? .dynamic-nft-market list-nft-fixed 
  'ST1NFTCONTRACT... 
  u42 
  u500000000 
  (some 'ST1CREATOR...) 
  u1000)

;; 2. Buyer purchases
(contract-call? .dynamic-nft-market buy-nft 'ST1NFTCONTRACT... u42)
;; Result: 487.5 STX to seller, 12.5 STX platform fee, 50 STX royalty
```

### Auction Flow

```clarity
;; 1. Create 7-day auction
(contract-call? .dynamic-nft-market list-nft-auction
  'ST1NFTCONTRACT...
  u42
  u100000000    ;; 100 STX min bid
  u500000000    ;; 500 STX reserve
  u1008         ;; ~7 days
  (some 'ST1CREATOR...)
  u500)         ;; 5% royalty

;; 2. User places bid
(contract-call? .dynamic-nft-market place-bid 'ST1NFTCONTRACT... u42 u600000000)

;; 3. Another user outbids (first user automatically refunded)
(contract-call? .dynamic-nft-market place-bid 'ST1NFTCONTRACT... u42 u700000000)

;; 4. After auction ends, anyone finalizes
(contract-call? .dynamic-nft-market finalize-auction 'ST1NFTCONTRACT... u42)
```

### Making and Accepting Offers

```clarity
;; 1. Buyer makes offer valid for 30 days
(contract-call? .dynamic-nft-market make-offer
  'ST1NFTCONTRACT...
  u42
  u450000000
  u4320)  ;; ~30 days from now

;; 2. Owner accepts offer
(contract-call? .dynamic-nft-market accept-offer 'ST1NFTCONTRACT... u42 'ST1BUYER...)
```

## Fee Structure

**Platform Fee**: 2.5% (default, configurable up to 10%)
**Creator Royalty**: Set per listing (up to 50%)

### Fee Distribution Example
Sale Price: 1000 STX
- Platform Fee (2.5%): 25 STX
- Royalty (10%): 100 STX
- Seller Receives: 875 STX

## Security Features

- Locked funds during auctions and offers
- Automatic refunds for outbid users
- Authorization checks on all sensitive operations
- State validation before transactions
- Maximum royalty caps (50%)
- Reserve price protection for auctions

## Error Codes

- `u200`: Owner-only operation
- `u201`: Listing/auction/offer not found
- `u202`: Unauthorized action
- `u203`: Invalid price or parameters
- `u204`: Listing already exists
- `u205`: Auction still active
- `u206`: Auction has ended
- `u207`: Bid too low
- `u208`: NFT not for sale

## Integration Guide

### For NFT Projects

1. Deploy your NFT contract
2. Register collection royalties (optional):
```clarity
(contract-call? .dynamic-nft-market register-collection-royalty
  .your-nft-contract
  'ST1CREATOR...
  u1000)  ;; 10%
```

### For Marketplace Frontends

1. Query listings: `get-listing`
2. Check auction status: `get-auction`
3. Display offer history per NFT
4. Show sales history for price discovery

## Best Practices

- Set reasonable reserve prices for auctions
- Use appropriate auction durations (minimum 1 day)
- Verify NFT ownership before listing
- Check offer expiration before accepting
- Monitor auction end times
- Set fair royalty percentages (5-10% typical)

## Deployment

```bash
clarinet contract deploy dynamic-nft-market
```

## Testing

```bash
clarinet test
```

## Future Enhancements

- Bundle sales (multiple NFTs)
- Dutch auctions (declining price)
- Lazy minting integration
- Collection-wide offers
- Whitelist/private sales

## License

MIT License
