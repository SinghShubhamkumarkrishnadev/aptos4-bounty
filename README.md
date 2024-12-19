---
# NFT Marketplace Smart Contract
---

## Introduction

The NFT Marketplace allows users to:
- Mint NFTs with metadata, rarity, and price.
- Buy, sell, and transfer NFTs.
- Like and tip NFTs.
- Manage and view offers on NFTs.
- Query marketplace data efficiently.

This contract ensures secure ownership, transaction integrity, and seamless user experience with a transparent fee structure.

---

## Key Components

### NFT Structure

Defines the properties of an NFT:
- `id`: Unique identifier.
- `owner`: Current owner of the NFT.
- `creator`: Address of the NFT creator.
- `name`: Name of the NFT.
- `description`: Description of the NFT.
- `uri`: Metadata URI for the NFT.
- `price`: Listing price in coins.
- `for_sale`: Boolean indicating sale status.
- `offers`: List of offers from potential buyers.
- `rarity`: Rarity level of the NFT (e.g., 1-10).
- `likes`: Number of likes the NFT has received.
- `likers`: Addresses of users who liked the NFT.

### Marketplace Structure

Holds the list of all NFTs:
- `nfts`: A vector containing all NFTs in the marketplace.

### ListedNFT Structure

Represents NFTs listed for sale:
- `id`: NFT identifier.
- `price`: Listing price.
- `rarity`: Rarity level.

### Constants

- `MARKETPLACE_FEE_PERCENT`: 2% transaction fee for marketplace operations.
- `TIP_FEE_PERCENT`: 1% fee on tips or donations.
- `FEE_COLLECTION_ADDRESS`: Address where marketplace fees are collected.

---

## Core Functionalities

### Initialization and Setup

- **Initialize Marketplace**: Sets up an empty marketplace structure.
- **Check Initialization**: Confirms if the marketplace is initialized.

### Minting NFTs

- **Mint New NFT**: Allows creators to mint NFTs. Optional minting fee is applied for non-whitelisted users.

### Buying and Selling NFTs

- **List for Sale**: Allows NFT owners to list their assets for sale at a specified price.
- **Set Price**: Updates the price of an NFT.
- **Purchase NFT**: Enables users to buy listed NFTs. Includes marketplace fee deduction.

### Offers Management

- **Make Offer**: Submit an offer for an NFT.
- **Accept Offer**: Owner accepts an offer, transferring ownership to the buyer.
- **Decline Offer**: Reject a specific offer from a buyer.
- **View Offers**: Retrieve all active offers for an NFT.

### Liking and Tipping NFTs

- **Like NFT**: Users can like an NFT, rewarding the creator with a fee. Duplicate likes are not allowed.
- **Tip or Donate**: Directly send coins to an NFT creator as a tip or donation.

### NFT Queries

- **Get NFT Details**: Fetch metadata and status of a specific NFT.
- **Retrieve NFTs for Sale**: List all NFTs currently available for sale.
- **Retrieve NFTs by Rarity**: Query NFTs based on rarity level.
- **Retrieve NFTs for Owner**: Fetch NFTs owned by a specific user.
- **Check Sale Status**: Verify if an NFT is listed for sale.
- **Get Price**: Retrieve the current price of an NFT.
- **Get Likes and Likers**: Fetch the like count and list of users who liked an NFT.

---

## Helper Functions

- **`check_whitelist`**: Verifies if a user is whitelisted to bypass minting fees.
- **`min`**: Returns the minimum value between two numbers.

---

## Error Handling

The contract employs robust error handling with specific error codes:
- **800**: Duplicate like attempt.
- **801**: Insufficient balance for liking.
- **500-706**: Various errors for invalid transactions, unauthorized actions, and data mismatches.

---
# New NFT Marketplace Features
---

## 1. **NFT Transfers**

### Description:
This functionality allows users to directly transfer ownership of their NFTs to other wallets via the marketplace. It facilitates gifting or trading of NFTs without requiring additional interactions outside the marketplace.

### Functionality:

#### Key Points:
- Ensures the sender is the current owner of the NFT.
- Prevents transfers to the current owner.
- Resets the `for_sale` status and price of the NFT after the transfer.

#### Code Snippet:
```move
public entry fun transfer_nft(
    account: &signer,
    marketplace_addr: address,
    nft_id: u64,
    recipient: address
) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
    let nft_ref = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft_ref.owner == signer::address_of(account), 500); // Caller is not the owner
    assert!(nft_ref.owner != recipient, 501); // Prevent transfer to the same owner

    // Update NFT ownership and reset its for_sale status and price
    nft_ref.owner = recipient;
    nft_ref.for_sale = false;
    nft_ref.price = 0;
}
```

---

## 2. **Offer System**

### Description:
This feature allows users to make, accept, or decline offers on NFTs listed in the marketplace. It enhances trading by enabling direct negotiations between buyers and sellers.

### Functionality:

#### Key Points:
- Buyers can propose a price for NFTs, even if they are not listed for sale.
- Sellers can review and accept offers or decline them outright.

#### Code Snippets:

**Make an Offer:**
```move
public entry fun make_offer(
    account: &signer,
    marketplace_addr: address,
    nft_id: u64,
    offer_price: u64
) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(offer_price > 0, 700); // Offer price must be positive
    assert!(nft.owner != signer::address_of(account), 701); // Owner cannot make an offer

    let offer = Offer {
        buyer: signer::address_of(account),
        offer_price,
    };

    vector::push_back(&mut nft.offers, offer);
}
```

**Accept an Offer:**
```move
public entry fun accept_offer(
    account: &signer,
    marketplace_addr: address,
    nft_id: u64,
    buyer: address
) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft.owner == signer::address_of(account), 702); // Only the owner can accept offers
    assert!(nft.for_sale, 703); // NFT must be listed for sale

    let offer_index: u64 = 0;
    let offers_len = vector::length<Offer>(&nft.offers);

    while (offer_index < offers_len) {
        let offer = vector::borrow<Offer>(&nft.offers, offer_index);
        if (offer.buyer == buyer) {
            let offer_price = offer.offer_price;
            let marketplace_fee = offer_price / 50; // 2% fee
            let seller_revenue = offer_price - marketplace_fee;

            coin::transfer<aptos_coin::AptosCoin>(account, nft.owner, seller_revenue);
            coin::transfer<aptos_coin::AptosCoin>(account, marketplace_addr, marketplace_fee);

            nft.owner = buyer;
            nft.for_sale = false;
            nft.price = 0;
            nft.offers = vector::empty<Offer>();
            return;
        };
        offer_index = offer_index + 1;
    };

    assert!(false, 704); // No matching offer found
}
```

**Decline an Offer:**
```move
public entry fun decline_offer(
    account: &signer,
    marketplace_addr: address,
    nft_id: u64,
    buyer: address
) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    assert!(nft.owner == signer::address_of(account), 705); // Only the owner can decline offers

    let i: u64 = 0;
    let offers_len = vector::length<Offer>(&nft.offers);

    while (i < offers_len) {
        let offer = vector::borrow<Offer>(&nft.offers, i);
        if (offer.buyer == buyer) {
            vector::remove(&mut nft.offers, i);
            return;
        };
        i = i + 1;
    };
    assert!(false, 706); // Offer not found
}
```

---

## 3. **APT Payments**

### Description:
Incorporates APT tokens as the primary medium of exchange in the marketplace. Includes tipping or donation functionality to support creators.

### Functionality:

#### Key Points:
- Buyers pay for NFTs and tips/donations using APT tokens.
- Ensures secure payment and fee deduction for transactions.

#### Code Snippet:
**Tip or Donate:**
```move
public entry fun tip_or_donate(
    account: &signer,
    marketplace_addr: address,
    nft_id: u64,
    amount: u64
) acquires Marketplace {
    let marketplace = borrow_global<Marketplace>(marketplace_addr);
    let nft = vector::borrow(&marketplace.nfts, nft_id);

    assert!(amount > 0, 600); // Invalid donation amount

    let tip_fee = (amount * TIP_FEE_PERCENT) / 100;
    let creator_revenue = amount - tip_fee;

    coin::transfer<aptos_coin::AptosCoin>(account, nft.creator, creator_revenue);
    coin::transfer<aptos_coin::AptosCoin>(account, marketplace_addr, tip_fee);
}
```

---

## 4. **Minting for All Users**

### Description:
Allows all connected users to mint NFTs. Introduces optional minting fees and a whitelist for free minting.

### Functionality:

#### Key Points:
- Whitelisted users can mint for free.
- Non-whitelisted users must pay a minting fee.

#### Code Snippet:
```move
public entry fun mint_nft(
    account: &signer, 
    name: vector<u8>, 
    description: vector<u8>, 
    uri: vector<u8>, 
    rarity: u8, 
    fee: u64,
    whitelist: vector<address>
) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(signer::address_of(account));
    let nft_id = vector::length(&marketplace.nfts);

    let user_addr = signer::address_of(account);
    let is_whitelisted = check_whitelist(&whitelist, user_addr);

    if (!is_whitelisted) {
        assert!(coin::balance<aptos_coin::AptosCoin>(user_addr) >= fee, 500); // Insufficient balance
        coin::transfer<aptos_coin::AptosCoin>(account, FEE_COLLECTION_ADDRESS, fee);
    };

    let new_nft = NFT {
        id: nft_id,
        owner: user_addr,
        creator: user_addr,
        name,
        description,
        uri,
        price: 0,
        for_sale: false,
        rarity,
        offers: vector::empty(), 
        likes: 0, 
        likers: vector::empty<address>(),
    };

    vector::push_back(&mut marketplace.nfts, new_nft);
}
```

**Helper Function:**
```move
fun check_whitelist(whitelist: &vector<address>, user: address): bool {
    let whitelist_len = vector::length(whitelist);
    let i = 0;
    while (i < whitelist_len) {
        if (*vector::borrow(whitelist, i) == user) {
            return true;
        };
        i = i + 1;
    };
    false
}
```
---

## 5. **Like Functionality with Fee and Liker Tracking**

### Description:
The Like NFT feature allows users to like NFTs, rewarding creators with a like fee and tracking unique likers.

### Key Features:
1. **Like Fee:**
   - Users are required to pay a fee to like an NFT.
   - The fee is transferred directly to the NFT creator.

2. **Duplicate Like Prevention:**
   - Users cannot like the same NFT more than once.
   - Checks are performed to ensure that a liker isn't already in the `likers` list.

3. **View Functions:**
   - Users can query the total number of likes and the list of likers for a specific NFT.

---

#### Code Snippet:

#### 1. **Like an NFT:**
```move
public entry fun like_nft(
    account: &signer,
    marketplace_addr: address,
    nft_id: u64,
    like_fee: u64
) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(marketplace_addr);
    let nft = vector::borrow_mut(&mut marketplace.nfts, nft_id);

    let liker = signer::address_of(account);

    // Ensure the liker hasn't already liked the NFT
    let likers_len = vector::length(&nft.likers);
    let i = 0;
    while (i < likers_len) {
        if (*vector::borrow(&nft.likers, i) == liker) {
            assert!(false, 800); // Duplicate like not allowed
        };
        i = i + 1;
    };

    // Transfer the like fee to the NFT creator
    assert!(coin::balance<aptos_coin::AptosCoin>(liker) >= like_fee, 801); // Insufficient balance
    coin::transfer<aptos_coin::AptosCoin>(account, nft.creator, like_fee);

    // Increment like count and add liker to the list
    nft.likes = nft.likes + 1;
    vector::push_back(&mut nft.likers, liker);
}
```

---

#### 2. **Get Like Count:**
```move
#[view]
public fun get_like_count(
    marketplace_addr: address,
    nft_id: u64
): u64 acquires Marketplace {
    let marketplace = borrow_global<Marketplace>(marketplace_addr);
    let nft = vector::borrow(&marketplace.nfts, nft_id);
    nft.likes
}
```


---

#### 3. **Get Likers:**
```move
#[view]
public fun get_likers(
    marketplace_addr: address,
    nft_id: u64
): vector<address> acquires Marketplace {
    let marketplace = borrow_global<Marketplace>(marketplace_addr);
    let nft = vector::borrow(&marketplace.nfts, nft_id);
    nft.likers
}
```

---
### 6. Advanced Filtering and Sorting Feature

### Description:  
The Advanced Filtering and Sorting feature allows users to refine their NFT searches based on various criteria and sort results for a better browsing experience.

### Functionality:  
Users can filter NFTs by attributes like price range and rarity, search by keywords, and sort by price or likes.

#### Key Points:
1. **Search Functionality:** Enables keyword-based search in NFT names and descriptions.
2. **Dynamic Filtering:** Filters by price range and rarity for tailored results.
3. **Sorting Options:** Sort NFTs by price (ascending/descending) or popularity (likes).
4. **Enhanced User Experience:** Simplifies navigation and improves discoverability of relevant NFTs.
5. **Customization:** Offers flexible criteria to suit diverse user preferences.

### Code Snippet:
```javascript
const applyFiltersAndSort = () => {
  let filteredNfts = [...nfts];

  // Search by name or description
  if (searchQuery) {
    filteredNfts = filteredNfts.filter(
      (nft) =>
        nft.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
        nft.description.toLowerCase().includes(searchQuery.toLowerCase())
    );
  }

  // Filter by rarity
  if (rarity !== 'all') {
    filteredNfts = filteredNfts.filter((nft) => nft.rarity === rarity);
  }

  // Sorting based on price or likes
  if (sortOption === 'priceAsc') {
    filteredNfts.sort((a, b) => a.price - b.price);
  } else if (sortOption === 'priceDesc') {
    filteredNfts.sort((a, b) => b.price - a.price);
  } else if (sortOption === 'likesDesc') {
    filteredNfts.sort((a, b) => b.likes - a.likes);
  }

  return filteredNfts;
};
```
--- 
