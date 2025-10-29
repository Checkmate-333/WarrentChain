# WarrentChain<img width="1919" height="1016" alt="WarrentChain" src="https://github.com/user-attachments/assets/6a759230-8ea5-4dbb-a3a5-b723177eebe5" />

Project Name - WARRENTCHAIN
<br>
Project Overview
   The project aims to create a secure and transparent system that stores and verifies product warranties immutably. Traditional warranty systems rely on paper or centralized digital storage, which can be lost, forged, or manipulated. This project eliminates such issues by recording warranty details on an immutable ledger.
Each product’s warranty data — including purchase date, serial number, and validity — is permanently stored and can be accessed anytime by customers, retailers, or service centers for verification. This ensures authenticity, transparency, and trust between brands and customers.
<br>
smart code contact link - https://repo.sourcify.dev/11142220/0xcd33Af4ca3f3883Fd58C5314545142B53f5cdFc9/
<br>
Usage Interactions -
1. Registration.
2. Customer Purchase.
3.  Verification.
4.  Claim Process.
   
USED CODE -

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;


contract WarrentChain {
    /// @notice Warranty record
    struct Warranty {
        uint256 id;             // warranty id (monotonic)
        address issuer;         // who issued the warranty (brand / seller)
        address owner;          // owner/buyer of the product
        string productId;       // product identifier (SKU, serial, or UPC)
        string productModel;    // human-friendly product name/model
        uint256 purchaseDate;   // unix timestamp of purchase
        uint256 duration;       // warranty duration in seconds
        string termsHash;       // IPFS/sha256 hash of warranty terms or URL
        bool extended;          // whether this is an extended warranty (optional flag)
    }

    // --- storage ---
    uint256 private _nextId;
    address public contractOwner;
    mapping(uint256 => Warranty) private _warranties;
    mapping(address => uint256[]) private _warrantiesByOwner;
    mapping(address => bool) public authorizedIssuer;

    // --- events ---
    event WarrantyCreated(uint256 indexed id, address indexed issuer, address indexed owner, string productId);
    event IssuerAuthorized(address indexed issuer);
    event IssuerRevoked(address indexed issuer);
    event ContractOwnerTransferred(address indexed previousOwner, address indexed newOwner);

    // --- modifiers ---
    modifier onlyOwner() {
        require(msg.sender == contractOwner, "WarrentChain: caller is not the contract owner");
        _;
    }

    modifier onlyAuthorizedIssuer() {
        require(authorizedIssuer[msg.sender], "WarrentChain: caller is not an authorized issuer");
        _;
    }

    constructor() {
        contractOwner = msg.sender;
        _nextId = 1; // start ids at 1
        // contract deployer is an authorized issuer by default
        authorizedIssuer[msg.sender] = true;
        emit IssuerAuthorized(msg.sender);
    }

    // --- issuer management ---
    /// @notice Authorize an address to issue warranties
    function authorizeIssuer(address issuer) external onlyOwner {
        require(issuer != address(0), "WarrentChain: zero address");
        authorizedIssuer[issuer] = true;
        emit IssuerAuthorized(issuer);
    }

    /// @notice Revoke an issuer
    function revokeIssuer(address issuer) external onlyOwner {
        require(issuer != address(0), "WarrentChain: zero address");
        authorizedIssuer[issuer] = false;
        emit IssuerRevoked(issuer);
    }

    /// @notice Transfer contract ownership
    function transferContractOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "WarrentChain: new owner is the zero address");
        emit ContractOwnerTransferred(contractOwner, newOwner);
        contractOwner = newOwner;
    }

    // --- primary functionality ---
    /// @notice Create a new immutable warranty record
    /// @param owner The owner/buyer of the product
    /// @param productId Product serial/SKU/identifier
    /// @param productModel Human-friendly product model/name
    /// @param purchaseDate Unix timestamp of purchase
    /// @param duration Warranty duration in seconds
    /// @param termsHash IPFS hash / SHA256 / URL that points to the full warranty terms
    /// @param extended Optional flag whether warranty is extended
    /// @return id The id of the created warranty
    function createWarranty(
        address owner,
        string calldata productId,
        string calldata productModel,
        uint256 purchaseDate,
        uint256 duration,
        string calldata termsHash,
        bool extended
    ) external onlyAuthorizedIssuer returns (uint256 id) {
        require(owner != address(0), "WarrentChain: owner is zero address");
        require(duration > 0, "WarrentChain: duration must be > 0");

        id = _nextId++;
        Warranty memory w = Warranty({
            id: id,
            issuer: msg.sender,
            owner: owner,
            productId: productId,
            productModel: productModel,
            purchaseDate: purchaseDate,
            duration: duration,
            termsHash: termsHash,
            extended: extended
        });

        _warranties[id] = w;
        _warrantiesByOwner[owner].push(id);

        emit WarrantyCreated(id, msg.sender, owner, productId);
    }

    // --- view helpers ---
    /// @notice Get a warranty by id
    function getWarranty(uint256 id) external view returns (Warranty memory) {
        require(id > 0 && id < _nextId, "WarrentChain: invalid id");
        return _warranties[id];
    }

    /// @notice Returns warranty ids owned by an address
    function getWarrantyIdsByOwner(address owner) external view returns (uint256[] memory) {
        return _warrantiesByOwner[owner];
    }

    /// @notice Returns total number of warranties created
    function totalWarranties() external view returns (uint256) {
        return _nextId - 1;
    }

    /// @notice Check whether a warranty is currently active
    function isWarrantyActive(uint256 id) external view returns (bool) {
        require(id > 0 && id < _nextId, "WarrentChain: invalid id");
        Warranty storage w = _warranties[id];
        uint256 expiry = w.purchaseDate + w.duration;
        return block.timestamp <= expiry;
    }

    // --- safety & recovery ---
    /// @notice Emergency function to recover accidentally sent ETH
    function recoverETH(address payable to) external onlyOwner {
        require(to != address(0), "WarrentChain: zero address");
        to.transfer(address(this).balance);
    }

    receive() external payable {}

    /// @notice Fallback - do nothing
    fallback() external payable {}
}


