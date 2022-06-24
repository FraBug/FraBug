//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;


import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/interfaces/IERC2981.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract SkinnyMouseGenesis is
    ERC721,
    IERC2981,
    AccessControl,
    Ownable
{
    using Strings for uint256;

    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;
    bytes32 public constant MANAGER_ROLE = keccak256("MANAGER_ROLE");

    uint256 public constant MAX_SUPPLY = 800;
    string public _baseURIextended = "https://bafybeiahchad4tzxsgjbsa5hwific2runycyhycxzigejagcj56vgogwda.ipfs.dweb.link/allMetadata/";
    bytes32 public _merkleRoot = 0x7f9947af1470e7df017d480516fb72e1b515240c75cf5afacfd79d475f309f35;
    address payable public _royaltyWallet = payable(address(this));
    uint256 public percentage = 5; // 5%
    // Sale / Presale
    mapping(address => uint256) private tickets;
    bool public presaleActive = false;
    bool public saleActive = false;
    uint256 public constant ETH_PRICE = 0.0999 ether;
    uint256 public constant PRESALE_ETH_PRICE = 0.0555 ether;
    uint256 public constant MAX_MINT_COUNT = 10;
    uint256 public constant PRESALE_MAX_MINT_COUNT = 2;

    constructor()
        ERC721("", "")
    {
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        grantRole(MANAGER_ROLE, msg.sender);
    }

    function withdraw(address withdrawalAddress)
        external
        onlyRole(MANAGER_ROLE)
    {
        payable(withdrawalAddress).transfer(address(this).balance);
    }



    function setBaseURI(string memory baseURI_)
        external
        onlyRole(MANAGER_ROLE)
    {
        _baseURIextended = baseURI_;
    }

    function contractURI()
        external
        view
        returns (string memory)
    {
        return string(abi.encodePacked(_baseURIextended, "metadata.json"));
    }

    function maxSupply()
        external
        pure
        returns (uint256)
    {
        return MAX_SUPPLY;
    }

    function totalSupply()
        external
        view
        returns (uint256)
    {
        return _tokenIds.current();
    }

    function tokenURI(uint256 tokenId)
        public
        view
        virtual
        override
        returns (string memory)
    {
        require(tokenId <= _tokenIds.current(), "Nonexistent token");
        return string(abi.encodePacked(_baseURIextended, tokenId.toString(), ".json"));
    }

    function claimTicket(uint256 count)
        internal
    {
        uint256 claimed = tickets[msg.sender];
        require(claimed + count <= PRESALE_MAX_MINT_COUNT, "Can't mint more than allocated");
        tickets[msg.sender] = claimed + count;
    }

    function setPresaleActive(bool val)
        external
        onlyRole(MANAGER_ROLE) 
    {
        presaleActive = val;
    }

    function setSaleActive(bool val)
        external
        onlyRole(MANAGER_ROLE) 
    {
        saleActive = val;
    }

    function setMerkleRoot(bytes32 merkleRoot_)
        external
        onlyRole(MANAGER_ROLE) 
    {
        _merkleRoot = merkleRoot_;
    }

    function presaleMint(uint256 count, uint256 index, bytes32[] calldata proof)
        external
        payable
        returns (uint256)
    {
        require(presaleActive, "Presale has not begun");
        require((PRESALE_ETH_PRICE * count) == msg.value, "Incorrect ETH sent; check price!");
        require(_tokenIds.current() + count <= MAX_SUPPLY, "SOLD OUT");

        claimTicket(count);

        bytes32 leaf = keccak256(abi.encode(index, msg.sender));
        require(MerkleProof.verify(proof, _merkleRoot, leaf), "Invalid proof");

        for (uint256 i=0; i<count; i++) {
            _tokenIds.increment();
            _mint(msg.sender, _tokenIds.current());
        }
        return _tokenIds.current();
    }

    function transferOwnership(address _newOwner)
        public
        override
        onlyOwner
    {
        address currentOwner = owner();
        _transferOwnership(_newOwner);
        grantRole(MANAGER_ROLE, _newOwner);
        grantRole(DEFAULT_ADMIN_ROLE, _newOwner);
        revokeRole(MANAGER_ROLE, currentOwner);
        revokeRole(DEFAULT_ADMIN_ROLE, currentOwner);
    }

    function mint(uint256 count)
        external
        payable
        returns (uint256)
    {
        require(saleActive, "Sale has not begun");
        require((ETH_PRICE * count) >= msg.value, "Incorrect ETH sent; check price!");
        require(count <= MAX_MINT_COUNT, "Tried to mint too many NFTs at once");
        require(_tokenIds.current() + count <= MAX_SUPPLY, "SOLD OUT");
        for (uint256 i=0; i<count; i++) {
            _tokenIds.increment();
            _mint(msg.sender, _tokenIds.current());
        }
        return _tokenIds.current();
    }

    // Allows an admin to mint for free, and send it to an address
    // This can be run while the contract is paused
    function teamMint(uint256 count, address recipient)
        external
        onlyRole(MANAGER_ROLE)
    returns (uint256)
    {
        require(_tokenIds.current() + count <= MAX_SUPPLY, "SOLD OUT");
        for (uint256 i=0; i<count; i++) {
            _tokenIds.increment();
            _mint(recipient, _tokenIds.current());
        }
        return _tokenIds.current();
    }

    function setRoyaltyWallet(address payable royaltyWallet_)
        external
        onlyRole(MANAGER_ROLE)
    {
        _royaltyWallet = (royaltyWallet_);
    }

    /**
     * @dev See {IERC165-royaltyInfo}.
     */
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        external
        view
        override
        returns (address receiver, uint256 royaltyAmount)
    {
        require(_exists(tokenId), "Nonexistent token");
        return (payable(_royaltyWallet), uint((salePrice * percentage * 100)/10000));
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override(ERC721, IERC165, AccessControl)
        returns (bool)
    {
        return interfaceId == type(IERC2981).interfaceId || super.supportsInterface(interfaceId);
    }

    receive () external payable {}
    fallback () external payable {}
}
