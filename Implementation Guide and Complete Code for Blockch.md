# Implementation Guide and Complete Code for Blockchain-Enabled Content Management System (CMS) for Secure Web Publishing

This guide provides you with all essential code components and explanations to implement your **Blockchain-Enabled CMS**, covering wallet login, content hashing, NFT minting, offline signing, decentralized storage with IPFS, and immutable edit tracking.

## Table of Contents

1. Wallet Login with MetaMask
2. Content Hashing with SHA-256 / Keccak-256
3. IPFS Integration for Decentralized Storage
4. Smart Contract for Blog NFT, Content Hashing \& Edit Tracking
5. Frontend Code to Integrate Blockchain and IPFS
6. Offline Signing and Signature Verification
7. Deployment \& Recommended Tools

## 1. Wallet Login with MetaMask

Authenticate users securely without passwords by connecting their MetaMask wallet.

```javascript
import { useState } from 'react';
import { ethers } from 'ethers';

function WalletLogin() {
  const [address, setAddress] = useState('');

  async function connectWallet() {
    if (window.ethereum) {
      const provider = new ethers.BrowserProvider(window.ethereum);
      await provider.send("eth_requestAccounts", []);
      const signer = provider.getSigner();
      const addr = await signer.getAddress();
      setAddress(addr);
    } else {
      alert('Please install MetaMask to login.');
    }
  }

  return (
    <div>
      <button onClick={connectWallet}>Login with MetaMask</button>
      {address && <p>Connected Wallet: {address}</p>}
    </div>
  );
}

export default WalletLogin;
```


## 2. Content Hashing

Use **Keccak-256** (Ethereum standard) or **SHA-256** to generate content hashes for tamper-proof verification.

### JavaScript Example with Keccak-256

```javascript
import { keccak256 } from 'ethers';

function hashContent(content) {
  const utf8Bytes = new TextEncoder().encode(content);
  return keccak256(utf8Bytes);
}
```


### Alternative: SHA-256 using Web Crypto API

```javascript
async function hashSHA256(content) {
  const encoder = new TextEncoder();
  const data = encoder.encode(content);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}
```


## 3. IPFS Integration (Decentralized Storage)

Upload content to IPFS and get the content identifier (CID) for decentralized hosting.

```javascript
import { create } from 'ipfs-http-client';

const ipfs = create({ host: 'ipfs.infura.io', port: 5001, protocol: 'https' });

async function uploadToIPFS(content) {
  const { path } = await ipfs.add(content);
  return path;  // IPFS CID
}
```


## 4. Smart Contract (Solidity)

Smart contract for minting blog posts as NFTs, storing content/IPFS hashes, edit tracking, and verifying ownership.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract BlogCMS is ERC721 {
    using ECDSA for bytes32;

    struct BlogPost {
        string ipfsHash;
        bytes32 contentHash;
        uint256[] editTimestamps;
        bytes32[] editHashes;
        bytes signature;
    }

    mapping(uint256 => BlogPost) public posts;
    uint256 public postCount;

    event PostCreated(uint256 indexed postId, address indexed author, string ipfsHash);
    event PostEdited(uint256 indexed postId, bytes32 newContentHash, uint256 timestamp);

    constructor() ERC721("BlogCMS", "BCMS") {}

    function createPost(string memory _ipfsHash, bytes32 _contentHash, bytes memory _signature) public returns (uint256) {
        // Verify signature to confirm content authenticity
        bytes32 signedHash = _contentHash.toEthSignedMessageHash();
        require(signedHash.recover(_signature) == msg.sender, "Invalid signature");

        postCount++;
        posts[postCount] = BlogPost(_ipfsHash, _contentHash, new uint256[](0), new bytes32[](0), _signature);
        _safeMint(msg.sender, postCount);

        emit PostCreated(postCount, msg.sender, _ipfsHash);
        return postCount;
    }

    function editPost(uint256 _postId, string memory _newIpfsHash, bytes32 _newContentHash, bytes memory _newSignature) public {
        require(ownerOf(_postId) == msg.sender, "Only owner can edit");
        BlogPost storage post = posts[_postId];

        bytes32 signedHash = _newContentHash.toEthSignedMessageHash();
        require(signedHash.recover(_newSignature) == msg.sender, "Invalid signature");

        // Append edit history
        post.editTimestamps.push(block.timestamp);
        post.editHashes.push(post.contentHash);

        post.ipfsHash = _newIpfsHash;
        post.contentHash = _newContentHash;
        post.signature = _newSignature;

        emit PostEdited(_postId, _newContentHash, block.timestamp);
    }

    function verifyContent(uint256 _postId, string memory content) public view returns (bool) {
        return keccak256(abi.encodePacked(content)) == posts[_postId].contentHash;
    }
}
```


## 5. Frontend Integration: Create \& Mint Blog Post

```jsx
import { useState } from 'react';
import { ethers } from 'ethers';
import { create } from 'ipfs-http-client';
import BlogCMSAbi from './abi/BlogCMS.json';

const ipfs = create({ host: 'ipfs.infura.io', port: 5001, protocol: 'https' });
const CONTRACT_ADDRESS = "YOUR_CONTRACT_ADDRESS_HERE";

export default function CreateBlog() {
  const [content, setContent] = useState('');
  const [loading, setLoading] = useState(false);

  async function handlePublish() {
    if (!window.ethereum) {
      alert("Please install MetaMask");
      return;
    }

    setLoading(true);
    try {
      // Upload content to IPFS
      const { path } = await ipfs.add(content);

      // Hash content with keccak256
      const contentHash = ethers.keccak256(ethers.toUtf8Bytes(content));

      // Connect to Ethereum via MetaMask
      await window.ethereum.request({ method: 'eth_requestAccounts' });
      const provider = new ethers.BrowserProvider(window.ethereum);
      const signer = provider.getSigner();

      // Sign content hash for offline verification
      const signature = await signer.signMessage(ethers.toUtf8Bytes(content));

      // Interact with smart contract
      const contract = new ethers.Contract(CONTRACT_ADDRESS, BlogCMSAbi, signer);
      const tx = await contract.createPost(path, contentHash, signature);
      
      await tx.wait();

      alert('Blog post published successfully!');
    } catch (err) {
      console.error(err);
      alert('Publishing error, please try again.');
    } finally {
      setLoading(false);
      setContent('');
    }
  }

  return (
    <div>
      <textarea placeholder="Write your blog here..." value={content} onChange={e => setContent(e.target.value)} />
      <button disabled={loading || !content} onClick={handlePublish}>
        {loading ? "Publishing..." : "Publish Blog"}
      </button>
    </div>
  );
}
```


## 6. Offline Signing and Signature Verification

Users can sign blog content offline and submit the signed message to verify authorship.

```javascript
async function signContentOffline(content) {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const signer = provider.getSigner();
  const signature = await signer.signMessage(content);
  return signature;
}

function verifySignature(content, signature, expectedAddress) {
  const recoveredAddress = ethers.verifyMessage(content, signature);
  return recoveredAddress.toLowerCase() === expectedAddress.toLowerCase();
}
```


## 7. Deployment and Tools Recommendations

- **Smart Contract Deployment:** Use [Remix IDE](https://remix.ethereum.org/) or Hardhat framework to compile and deploy contracts to Ethereum testnets (Goerli, Mumbai).
- **Frontend Hosting:** Deploy frontend with Vercel, Netlify, or Fleek for static React apps.
- **Blockchain Access:** Use services like Infura or Alchemy for reliable Ethereum node access.
- **Wallet:** MetaMask browser extension for user authentication and transactions.
- **IPFS Pinning:** Pinata or Filebase to keep IPFS content persistently available.


## Summary

By integrating these components, your Blockchain-Enabled CMS will provide:

- Passwordless wallet-based login
- Tamper-proof blog posts via cryptographic hashes recorded on-chain
- NFT-based ownership of blog posts
- Immutable edit tracking with on-chain logs
- Decentralized storage for content with IPFS
- Offline content signing for secure, flexible publishing

These building blocks collectively create a secure, transparent, and decentralized CMS that leverages blockchain’s best features for web publishing.

If you want, I can also help with additional advanced features, testing scripts, or deployment automation. Just let me know!

