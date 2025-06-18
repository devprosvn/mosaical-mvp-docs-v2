# **NEXUS WEB3 LABS**

## **From: Dr. Eliza Nakamoto, Chief Blockchain Architect & Founder**

## **COMPREHENSIVE DEVELOPMENT BLUEPRINT: MOSAICAL NFT LENDING PLATFORM (UPDATED)**

Dear Mosaical Team,

This document provides an updated development blueprint for the Mosaical NFT lending platform, incorporating key technical changes as requested. The updates include transitioning the NFT API from Coingecko to Alchemy, integrating smart contract functionalities into the frontend using viem and wagmi, and migrating the frontend framework from React.js/TailwindCSS/PostCSS to Solid.js/Bootstrap. These changes aim to enhance the platform's capabilities, improve frontend performance and development experience, and align with the strategic pivot to the Saga Protocol ecosystem and GameFi market.

Below is a comprehensive breakdown of the POC, Prototype, and MVP phases with specific technical implementations, success metrics, and development considerations, reflecting the updated technology stack.

## **PROOF OF CONCEPT (POC) DETAILED BLUEPRINT (UPDATED)**

### **1\. Core Objectives**

The POC will demonstrate the fundamental technical feasibility of the GameFi-focused NFT lending model through a minimal implementation that validates:

* NFT collateralization mechanics on Saga Protocol  
* GameFi yield collection automation  
* Dynamic LTV adjustments based on GameFi NFT utility metrics  
* Basic partial liquidation functionality  
* Cross-chainlet NFT discoverability and pricing

### **2\. Technical Architecture (UPDATED)**

#### **2.1 Smart Contract Structure**

// Core contract architecture (No changes in core logic for POC, focusing on integration points)

contract NFTVault {  
    // Maps user addresses to their deposited NFTs  
    mapping(address \=\> mapping(address \=\> mapping(uint256 \=\> bool))) public deposits;  
      
    // Chainlet registry for GameFi NFT collections  
    mapping(address \=\> bool) public supportedChainlets;  
    mapping(address \=\> mapping(address \=\> bool)) public supportedCollections;  
      
    // Deposit NFT function  
    function depositNFT(address collection, uint256 tokenId) external {  
        require(isGameFiNFT(collection), "Not a supported GameFi NFT");  
        // Transfer NFT to vault  
        IERC721(collection).transferFrom(msg.sender, address(this), tokenId);  
        deposits\[msg.sender\]\[collection\]\[tokenId\] \= true;  
        emit NFTDeposited(msg.sender, collection, tokenId);  
    }  
      
    // Withdraw NFT function (with loan check)  
    function withdrawNFT(address collection, uint256 tokenId) external {  
        require(deposits\[msg.sender\]\[collection\]\[tokenId\], "Not your NFT");  
        require(loanManager.getLoanAmount(msg.sender, collection, tokenId) \== 0, "Loan exists");  
          
        deposits\[msg.sender\]\[collection\]\[tokenId\] \= false;  
        IERC721(collection).transferFrom(address(this), msg.sender, tokenId);  
        emit NFTWithdrawn(msg.sender, collection, tokenId);  
    }  
      
    // Check if NFT is from supported GameFi collection  
    function isGameFiNFT(address collection) public view returns (bool) {  
        address chainlet \= getChainletFromCollection(collection);  
        return supportedChainlets\[chainlet\] && supportedCollections\[chainlet\]\[collection\];  
    }  
      
    // Get chainlet from collection address (for POC, simplified implementation)  
    function getChainletFromCollection(address collection) internal view returns (address) {  
        // For POC: Extract chainlet from collection address pattern  
        // In practice, will use Saga Protocol chainlet registry  
        return address(uint160(collection) & 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF00000000);  
    }  
      
    // Admin function to add supported chainlet (governance controlled in future)  
    function addSupportedChainlet(address chainlet) external onlyAdmin {  
        supportedChainlets\[chainlet\] \= true;  
    }  
      
    // Admin function to add supported collection within a chainlet  
    function addSupportedCollection(address chainlet, address collection) external onlyAdmin {  
        require(supportedChainlets\[chainlet\], "Chainlet not supported");  
        supportedCollections\[chainlet\]\[collection\] \= true;  
    }  
}

contract LoanManager {  
    // Maps NFTs to loan amounts  
    mapping(address \=\> mapping(address \=\> mapping(uint256 \=\> uint256))) public loans;  
      
    // Oracle reference  
    IPriceOracle public oracle;  
      
    // GameFi utility multiplier (POC version)  
    mapping(address \=\> uint256) public gameFiUtilityMultiplier;  
      
    // Borrow against NFT  
    function borrow(address collection, uint256 tokenId, uint256 amount) external {  
        require(nftVault.deposits(msg.sender, collection, tokenId), "Not your NFT");  
          
        // Get NFT price from oracle  
        uint256 nftPrice \= oracle.getNFTPrice(collection, tokenId);  
          
        // Apply GameFi utility multiplier to base LTV (default 50%)  
        uint256 utilityFactor \= gameFiUtilityMultiplier\[collection\];  
        if (utilityFactor \== 0\) utilityFactor \= 100; // Default 100% (no adjustment)  
          
        // Calculate max LTV with utility adjustment  
        uint256 baseMaxLTV \= 50; // 50% base LTV  
        uint256 adjustedMaxLTV \= (baseMaxLTV \* utilityFactor) / 100;  
          
        // Cap at 70% max LTV for safety  
        if (adjustedMaxLTV \> 70\) adjustedMaxLTV \= 70;  
          
        uint256 maxLoan \= (nftPrice \* adjustedMaxLTV) / 100;  
        require(amount \<= maxLoan, "Exceeds LTV");  
          
        loans\[msg.sender\]\[collection\]\[tokenId\] \= amount;  
        IERC20(lendingToken).transfer(msg.sender, amount);  
        emit LoanCreated(msg.sender, collection, tokenId, amount);  
    }  
      
    // Repay loan  
    function repay(address collection, uint256 tokenId, uint256 amount) external {  
        uint256 loanAmount \= loans\[msg.sender\]\[collection\]\[tokenId\];  
        require(loanAmount \> 0, "No loan exists");  
          
        uint256 repayAmount \= amount \> loanAmount ? loanAmount : amount;  
        loans\[msg.sender\]\[collection\]\[tokenId\] \-= repayAmount;  
          
        IERC20(lendingToken).transferFrom(msg.sender, address(this), repayAmount);  
        emit LoanRepaid(msg.sender, collection, tokenId, repayAmount);  
    }  
      
    // Set GameFi utility multiplier (admin function for POC)  
    function setGameFiUtilityMultiplier(address collection, uint256 multiplier) external onlyAdmin {  
        require(multiplier \<= 140, "Multiplier too high"); // Max 140% (allows up to 70% LTV)  
        gameFiUtilityMultiplier\[collection\] \= multiplier;  
    }  
}

contract SimpleDPOToken is ERC20 {  
    // Maps NFT to total supply of DPO tokens  
    mapping(address \=\> mapping(uint256 \=\> uint256)) public nftTokenSupply;  
      
    // Mint DPO tokens for NFT  
    function mintDPOTokens(address user, address collection, uint256 tokenId) external onlyLoanManager {  
        uint256 tokenSupply \= 1000 \* (10\*\*18); // 1000 tokens per NFT  
        \_mint(user, tokenSupply);  
        nftTokenSupply\[collection\]\[tokenId\] \= tokenSupply;  
    }  
      
    // Burn DPO tokens during repayment  
    function burnDPOTokens(address user, address collection, uint256 tokenId, uint256 amount) external onlyLoanManager {  
        \_burn(user, amount);  
    }  
}

contract GameFiPriceOracle {  
    // Floor prices by collection  
    mapping(address \=\> uint256) public collectionFloorPrices;  
      
    // GameFi utility score by collection (e.g., rarity, level, in-game stats)  
    mapping(address \=\> mapping(uint256 \=\> uint256)) public nftUtilityScores;  
      
    // Set floor price (admin only in POC)  
    function setCollectionFloorPrice(address collection, uint256 price) external onlyAdmin {  
        collectionFloorPrices\[collection\] \= price;  
    }  
      
    // Set NFT utility score (admin only in POC)  
    function setNFTUtilityScore(address collection, uint256 tokenId, uint256 score) external onlyAdmin {  
        nftUtilityScores\[collection\]\[tokenId\] \= score;  
    }  
      
    // Get NFT price with utility adjustment  
    function getNFTPrice(address collection, uint256 tokenId) external view returns (uint256) {  
        uint256 floorPrice \= collectionFloorPrices\[collection\];  
        uint256 utilityScore \= nftUtilityScores\[collection\]\[tokenId\];  
          
        if (utilityScore \== 0\) return floorPrice; // Default to floor if no score  
          
        // Apply utility score as multiplier (baseline is 100\)  
        return (floorPrice \* utilityScore) / 100;  
    }  
}

contract YieldCollector {  
    // Simulate yield collection for POC  
    function collectYield(address collection, uint256 tokenId) external returns (uint256) {  
        // In POC, return mock yield amount based on GameFi utility  
        uint256 yieldAmount \= mockGameFiYieldCalculation(collection, tokenId);  
          
        // Transfer yield to loan manager for repayment  
        IERC20(yieldToken).transfer(loanManager, yieldAmount);  
        return yieldAmount;  
    }  
      
    // Mock GameFi yield calculation  
    function mockGameFiYieldCalculation(address collection, uint256 tokenId) internal view returns (uint256) {  
        // For POC: Base yield (1% of NFT value per month) \+ GameFi utility bonus  
        uint256 nftValue \= oracle.getNFTPrice(collection, tokenId);  
        uint256 baseYield \= (nftValue \* 1 \* timeElapsed) / (100 \* 30 days);  
          
        // GameFi bonus: Extra 0-3% based on utility score  
        uint256 utilityScore \= oracle.nftUtilityScores(collection, tokenId);  
        if (utilityScore \== 0\) utilityScore \= 100; // Default  
          
        uint256 bonusYield \= (baseYield \* (utilityScore \- 100)) / 100;  
        if (bonusYield \> (baseYield \* 3)) bonusYield \= baseYield \* 3; // Cap at 300% bonus  
          
        return baseYield \+ bonusYield;  
    }  
}

#### **2.2 Oracle Implementation (UPDATED - Using Alchemy NFT API)**

// Alchemy NFT API for NFT Price Data and Metadata
// Endpoints to be used:
// - getNFTsForOwner (to get NFTs owned by an address)
// - getNFTMetadata (to get metadata for a specific NFT)
// - getFloorPrice (to get floor price for a collection)
// - getContractMetadata (to get collection details)

class AlchemyNFTPriceOracle {
  constructor(apiKey, network) {
    this.apiKey = apiKey; 
    this.baseUrl = `https://${network}.g.alchemy.com/nft/v2`; 
  }

  async getNFTsForOwner(ownerAddress) {
    try {
      const response = await fetch(`${this.baseUrl}/${this.apiKey}/getNFTsForOwner?owner=${ownerAddress}`);
      const data = await response.json();
      return data;
    } catch (error) {
      console.error(`Error fetching NFTs for owner ${ownerAddress}:`, error);
      return null;
    }
  }

  async getNFTMetadata(contractAddress, tokenId) {
    try {
      const response = await fetch(`${this.baseUrl}/${this.apiKey}/getNFTMetadata?contractAddress=${contractAddress}&tokenId=${tokenId}`);
      const data = await response.json();
      return data;
    } catch (error) {
      console.error(`Error fetching NFT metadata for ${contractAddress}:${tokenId}:`, error);
      return null;
    }
  }

  async getFloorPrice(contractAddress) {
    try {
      const response = await fetch(`${this.baseUrl}/${this.apiKey}/getFloorPrice?contractAddress=${contractAddress}`);
      const data = await response.json();
      return data;
    } catch (error) {
      console.error(`Error fetching floor price for ${contractAddress}:`, error);
      return null;
    }
  }

  async getCollectionMetadata(contractAddress) {
    try {
      const response = await fetch(`${this.baseUrl}/${this.apiKey}/getContractMetadata?contractAddress=${contractAddress}`);
      const data = await response.json();
      return data;
    } catch (error) {
      console.error(`Error fetching collection metadata for ${contractAddress}:`, error);
      return null;
    }
  }
}

#### **2.3 GameFi Yield Simulation Framework**

// GameFi Yield Simulation Framework (No changes in core logic for POC)

class GameFiYieldSimulator {  
  constructor(chainlets, collections, timeframe \= 90) { // 90 days simulation  
    this.chainlets \= chainlets;  
    this.collections \= collections;  
    this.timeframe \= timeframe;  
    this.dailyYields \= {};  
    this.volatilityFactors \= {};  
    this.utilityImpact \= {};  
    this.gameEngagement \= {};  
  }  
    
  // Initialize with historical data  
  async initialize() {  
    for (const chainlet of this.chainlets) {  
      for (const collection of this.collections\[chainlet\] || \[\]) {  
        const key \= \`${chainlet}:${collection}\`;  
          
        // Load historical yield data for collection  
        const historicalYield \= await this.fetchHistoricalYield(chainlet, collection);  
          
        // Calculate average daily yield  
        this.dailyYields\[key\] \= this.calculateAverageDailyYield(historicalYield);  
          
        // Calculate volatility factor  
        this.volatilityFactors\[key\] \= this.calculateVolatility(historicalYield);  
          
        // Calculate utility impact on yield  
        this.utilityImpact\[key\] \= await this.analyzeUtilityImpact(chainlet, collection);  
          
        // Calculate game engagement metrics (active users, playtime)  
        this.gameEngagement\[key\] \= await this.fetchGameEngagement(chainlet, collection);  
      }  
    }  
  }  
    
  // Analyze how NFT utility affects yield  
  async analyzeUtilityImpact(chainlet, collection) {  
    try {  
      // Fetch sample of NFTs with different utility scores  
      const nftSamples \= await this.fetchNFTSamplesWithUtility(chainlet, collection, 20);  
        
      // Calculate correlation between utility and yield  
      let totalCorrelation \= 0;  
      let sampleCount \= 0;  
        
      for (const nft of nftSamples) {  
        const yieldHistory \= await this.fetchNFTYieldHistory(chainlet, collection, nft.tokenId);  
        if (yieldHistory.length \> 0) {  
          const avgYield \= yieldHistory.reduce((sum, y) \=\> sum \+ y.yield, 0) / yieldHistory.length;  
          const normalizedYield \= avgYield / this.dailyYields\[\`${chainlet}:${collection}\`\];  
          const utilityFactor \= nft.utilityScore / 100;  
            
          // How closely utility predicts yield (1.0 would be perfect correlation)  
          const correlation \= normalizedYield / utilityFactor;  
          totalCorrelation \+= correlation;  
          sampleCount++;  
        }  
      }  
        
      return sampleCount \> 0 ? totalCorrelation / sampleCount : 1.0;  
    } catch (error) {  
      console.error("Error analyzing utility impact:", error);  
      return 1.0; // Default: utility has 1:1 impact on yield  
    }  
  }  
    
  // Run simulation with different market scenarios  
  async runSimulation(marketScenario \= \'normal\', gameActivity \= \'normal\') {  
    const results \= {};  
      
    for (const chainlet of this.chainlets) {  
      results\[chainlet\] \= {};  
        
      for (const collection of this.collections\[chainlet\] || \[\]) {  
        const key \= \`${chainlet}:${collection}\`;  
        const dailyYield \= this.dailyYields\[key\];  
        const volatility \= this.volatilityFactors\[key\];  
        const utilityImpact \= this.utilityImpact\[key\];  
          
        // Adjust for market scenario  
        let marketMultiplier;  
        switch (marketScenario) {  
          case \'bull\': marketMultiplier \= 1.5; break;  
          case \'bear\': marketMultiplier \= 0.6; break;  
          default: marketMultiplier \= 1.0; // normal  
        }  
          
        // Adjust for game activity level  
        let activityMultiplier;  
        switch (gameActivity) {  
          case \'viral\': activityMultiplier \= 1.8; break;  
          case \'growing\': activityMultiplier \= 1.3; break;  
          case \'declining\': activityMultiplier \= 0.7; break;  
          case \'inactive\': activityMultiplier \= 0.3; break;  
          default: activityMultiplier \= 1.0; // normal  
        }  
          
        // Simulate daily yields  
        const simulatedYields \= \[\];  
        let cumulativeYield \= 0;  
          
        for (let day \= 1; day \<= this.timeframe; day++) {  
          // Add randomness based on volatility  
          const randomFactor \= 1 \+ (Math.random() \* 2 \- 1) \* volatility;  
            
          // Final daily yield with all factors  
          const dailySimulatedYield \=   
            dailyYield \*   
            marketMultiplier \*   
            activityMultiplier \*   
            randomFactor \*   
            utilityImpact;  
            
          cumulativeYield \+= dailySimulatedYield;  
          simulatedYields.push({  
            day,  
            dailyYield: dailySimulatedYield,  
            cumulativeYield  
          });  
        }  
          
        results\[chainlet\]\[collection\] \= {  
          averageDailyYield: dailyYield \* marketMultiplier \* activityMultiplier \* utilityImpact,  
          totalYield: cumulativeYield,  
          yieldTimeSeries: simulatedYields,  
          annualizedYield: (cumulativeYield / this.timeframe) \* 365,  
          gameEngagementLevel: this.gameEngagement\[key\]  
        };  
      }  
    }  
      
    return results;  
  }  
}

### **3\. Testing Framework (UPDATED)**

#### **3.1 Smart Contract Testing**

// Example test cases for NFT Vault with Saga Protocol integration (No changes in core logic for POC)

describe("NFTVault on Saga Protocol", function() {  
  let nftVault, mockNFT, owner, user1;  
  let mockChainletId, mockCollectionAddress;  
    
  beforeEach(async function() {  
    // Deploy mock Saga Chainlet registry  
    const MockSagaRegistry \= await ethers.getContractFactory("MockSagaRegistry");  
    mockSagaRegistry \= await MockSagaRegistry.deploy();  
      
    // Register mock GameFi chainlet  
    mockChainletId \= "0x1234567890123456789012345678901234567890";  
    await mockSagaRegistry.registerChainlet(mockChainletId, "MockGameChainlet");  
      
    // Deploy mock NFT contract to represent a GameFi NFT  
    const MockNFT \= await ethers.getContractFactory("MockERC721");  
    mockNFT \= await MockNFT.deploy("MockGameNFT", "MGAME");  
    mockCollectionAddress \= mockNFT.address;  
      
    // Register mock collection in chainlet  
    await mockSagaRegistry.registerCollection(mockChainletId, mockCollectionAddress);  
      
    // Deploy NFT Vault  
    const NFTVault \= await ethers.getContractFactory("NFTVault");  
    nftVault \= await NFTVault.deploy(mockSagaRegistry.address);  
      
    \[owner, user1\] \= await ethers.getSigners();  
      
    // Mint NFT to user1  
    await mockNFT.connect(owner).mint(user1.address, 1);  
      
    // Add supported chainlet and collection to vault  
    await nftVault.connect(owner).addSupportedChainlet(mockChainletId);  
    await nftVault.connect(owner).addSupportedCollection(mockChainletId, mockCollectionAddress);  
  });  
    
  it("Should allow users to deposit GameFi NFTs from supported chainlets", async function() {  
    // Approve vault to transfer NFT  
    await mockNFT.connect(user1).approve(nftVault.address, 1);  
      
    // Deposit NFT  
    await nftVault.connect(user1).depositNFT(mockCollectionAddress, 1);  
      
    // Check deposit status  
    expect(await nftVault.deposits(user1.address, mockCollectionAddress, 1)).to.be.true;  
      
    // Check NFT ownership  
    expect(await mockNFT.ownerOf(1)).to.equal(nftVault.address);  
  });  
    
  it("Should reject NFT deposits from unsupported chainlets", async function() {  
    // Create new NFT collection not in supported list  
    const UnsupportedNFT \= await ethers.getContractFactory("MockERC721");  
    const unsupportedNFT \= await UnsupportedNFT.deploy("UnsupportedNFT", "UNSUP");  
      
    // Mint to user1  
    await unsupportedNFT.connect(owner).mint(user1.address, 1);  
    await unsupportedNFT.connect(user1).approve(nftVault.address, 1);  
      
    // Attempt deposit should fail  
    await expect(  
      nftVault.connect(user1).depositNFT(unsupportedNFT.address, 1)  
    ).to.be.revertedWith("Not a supported GameFi NFT");  
  });  
    
  it("Should allow users to withdraw NFTs if no loan exists", async function() {  
    // Setup: Deposit NFT  
    await mockNFT.connect(user1).approve(nftVault.address, 1);  
    await nftVault.connect(user1).depositNFT(mockCollectionAddress, 1);  
      
    // Withdraw NFT  
    await nftVault.connect(user1).withdrawNFT(mockCollectionAddress, 1);  
      
    // Check deposit status  
    expect(await nftVault.deposits(user1.address, mockCollectionAddress, 1)).to.be.false;  
      
    // Check NFT ownership  
    expect(await mockNFT.ownerOf(1)).to.equal(user1.address);  
  });  
});

// Example test cases for Loan Manager with GameFi utility (No changes in core logic for POC)

describe("LoanManager with GameFi Utility", function() {  
  // Similar setup code...  
    
  it("Should adjust LTV based on GameFi utility multiplier", async function() {  
    // Setup: Deposit NFT  
    await mockNFT.connect(user1).approve(nftVault.address, 1);  
    await nftVault.connect(user1).depositNFT(mockCollectionAddress, 1);  
      
    // Set NFT price in oracle  
    await mockOracle.setCollectionFloorPrice(mockCollectionAddress, ethers.utils.parseEther("100"));  
      
    // Set GameFi utility multiplier to 120% (allows 60% LTV instead of base 50%)  
    await loanManager.connect(owner).setGameFiUtilityMultiplier(mockCollectionAddress, 120);  
      
    // Borrow 60 ETH against 100 ETH NFT (should succeed with utility bonus)  
    await loanManager.connect(user1).borrow(  
      mockCollectionAddress,   
      1,   
      ethers.utils.parseEther("60")  
    );  
      
    // Check loan amount  
    expect(await loanManager.loans(user1.address, mockCollectionAddress, 1))  
      .to.equal(ethers.utils.parseEther("60"));  
  });  
    
  it("Should cap LTV at maximum 70% even with high utility", async function() {  
    // Setup similar to above  
      
    // Set high utility multiplier (150%)  
    await loanManager.connect(owner).setGameFiUtilityMultiplier(mockCollectionAddress, 150);  
      
    // Try to borrow 75 ETH against 100 ETH NFT (should fail as max is 70%)  
    await expect(  
      loanManager.connect(user1).borrow(  
        mockCollectionAddress,   
        1,   
        ethers.utils.parseEther("75")  
      )  
    ).to.be.revertedWith("Exceeds LTV");  
      
    // Borrow at max allowed LTV (70%)  
    await loanManager.connect(user1).borrow(  
      mockCollectionAddress,   
      1,   
      ethers.utils.parseEther("70")  
    );  
      
    // Check loan amount  
    expect(await loanManager.loans(user1.address, mockCollectionAddress, 1))  
      .to.equal(ethers.utils.parseEther("70"));  
  });  
});

#### **3.2 Oracle Testing (UPDATED - Using Alchemy NFT API)**

describe("GameFi Oracle Testing", function() {  
  let oracle;  
  const mockChainletId \= "0x1234567890123456789012345678901234567890";  
  const mockCollectionAddress \= "0xabcdef1234567890abcdef1234567890abcdef12";  
    
  beforeEach(async function() {  
    // Initialize oracle with mock API key and network  
    oracle \= new AlchemyNFTPriceOracle("mockApiKey", "eth-mainnet");  
      
    // Mock Alchemy API responses for testing  
    oracle.getFloorPrice \= async (collection) \=\> {  
      if (collection \=== mockCollectionAddress) {  
        return { floorPrice: { openSea: 100, looksRare: 102 } }; // Mock response structure  
      } else {  
        return null;  
      }  
    };  
      
    oracle.getNFTMetadata \= async (collection, tokenId) \=\> {  
        // Mock metadata response including attributes for utility score calculation
        return {
            metadata: {
                attributes: [
                    { trait_type: "level", value: 75 },
                    { trait_type: "rarity", value: "epic" },
                    { trait_type: "power", value: 850 }
                ]
            }
        };
    };

    // Mock utility calculation logic within the test or a helper function
    const calculateRpgUtilityScore = (attributes) => {
        let level = 0;
        let rarity = "common";
        let power = 0;

        for (const attr of attributes) {
            if (attr.trait_type === "level") level = attr.value;
            if (attr.trait_type === "rarity") rarity = attr.value;
            if (attr.trait_type === "power") power = attr.value;
        }

        const rarityBonus = {common: 0, uncommon: 5, rare: 15, epic: 30, legendary: 50}[rarity.toLowerCase()] || 0;
        return 100 + (level / 2) + rarityBonus + (power / 50);
    };

    // Override getNFTPrice to use mocked Alchemy calls and local utility calculation
    oracle.getNFTPrice = async (collection, tokenId) => {
        const floorData = await oracle.getFloorPrice(collection);
        const floorPrice = floorData ? floorData.floorPrice.openSea || floorData.floorPrice.looksRare || 0 : 0;

        const metadataResponse = await oracle.getNFTMetadata(collection, tokenId);
        const attributes = metadataResponse && metadataResponse.metadata && metadataResponse.metadata.attributes ? metadataResponse.metadata.attributes : [];
        const utilityScore = calculateRpgUtilityScore(attributes);

        if (utilityScore === 0) return floorPrice; // Default to floor if no score

        // Apply utility score as multiplier (baseline is 100)
        return (floorPrice * utilityScore) / 100;
    };

  });  
    
  it("Should fetch floor price from Alchemy", async function() {  
    const floorData \= await oracle.getFloorPrice(mockCollectionAddress);  
    expect(floorData).to.have.property("floorPrice");
    expect(floorData.floorPrice).to.have.property("openSea");
    expect(floorData.floorPrice.openSea).to.equal(100);
  });  
    
  it("Should fetch NFT metadata from Alchemy", async function() {
    const metadataResponse = await oracle.getNFTMetadata(mockCollectionAddress, 123);
    expect(metadataResponse).to.have.property("metadata");
    expect(metadataResponse.metadata).to.have.property("attributes");
    expect(metadataResponse.metadata.attributes).to.be.an("array");
  });

  it("Should calculate utility scores based on Alchemy metadata", async function() {  
    const utilityScore \= await oracle.getNFTPrice(  
      mockCollectionAddress,   
      123 // Token ID  
    );
      
    // Expected utility score calculation based on mocked metadata:
    // level: 75, rarity: "epic", power: 850
    // 100 + (75 / 2) + 30 + (850 / 50) = 100 + 37.5 + 30 + 17 = 184.5
    // Price calculation: floorPrice * utilityScore / 100 = 100 * 184.5 / 100 = 184.5
    expect(utilityScore).to.be.approximately(184.5, 0.1);  
  });  
});

#### **3.3 GameFi Yield Simulation Testing**

describe("GameFi Yield Simulation", function() {  
  let simulator;  
  const mockChainletId \= "0x1234567890123456789012345678901234567890";  
  const mockCollectionAddress \= "0xabcdef1234567890abcdef1234567890abcdef12";  
    
  beforeEach(async function() {  
    // Initialize simulator with test collections  
    simulator \= new GameFiYieldSimulator(  
      \[mockChainletId\],   
      {\[mockChainletId\]: \[mockCollectionAddress\]}  
    );  
      
    // Mock historical data fetching  
    simulator.fetchHistoricalYield \= async (chainlet, collection) \=\> {  
      if (collection \=== mockCollectionAddress) {  
        return Array(90).fill().map((\_, i) \=\> ({   
          day: i \+ 1,   
          yield: 0.002 \* (1 \+ 0.15 \* Math.sin(i / 8)) // 0.2% daily with sine wave variation  
        }));  
      } else {  
        return \[\];  
      }  
    };  
      
    // Mock utility impact analysis  
    simulator.fetchNFTSamplesWithUtility \= async () \=\> {  
      return Array(20).fill().map((\_, i) \=\> ({  
        tokenId: i \+ 1,  
        utilityScore: 100 \+ (i \* 5) // 100 to 195  
      }));  
    };  
      
    simulator.fetchNFTYieldHistory \= async (\_, \_\_, tokenId) \=\> {  
      const baseYield \= 0.002;  
      const utilityFactor \= tokenId % 20 ? (tokenId % 20) / 10 : 1; // 0.1 to 1.9  
      return Array(30).fill().map((\_, i) \=\> ({  
        day: i \+ 1,  
        yield: baseYield \* utilityFactor \* (1 \+ 0.1 \* Math.sin(i / 5))  
      }));  
    };  
      
    // Mock game engagement data  
    simulator.fetchGameEngagement \= async () \=\> ({  
      dailyActiveUsers: 15000,  
      averagePlaytime: 45, // minutes  
      retentionRate: 0.72,  
      monthlyRevenue: 320000  
    });  
      
    await simulator.initialize();  
  });  
    
  it("Should calculate accurate average yields", function() {  
    const key \= \`${mockChainletId}:${mockCollectionAddress}\`;  
    expect(simulator.dailyYields\[key\]).to.be.approximately(0.002, 0.0003);  
  });  
    
  it("Should calculate utility impact on yield", function() {  
    const key \= \`${mockChainletId}:${mockCollectionAddress}\`;  
    // Should be close to 1.0 if utility correctly predicts yield  
    expect(simulator.utilityImpact\[key\]).to.be.approximately(1.0, 0.2);  
  });  
    
  it("Should simulate different game activity scenarios", async function() {  
    // Normal game activity  
    const normalResults \= await simulator.runSimulation(\'normal\', \'normal\');  
    const normalYield \= normalResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
      
    // Viral game activity  
    const viralResults \= await simulator.runSimulation(\'normal\', \'viral\');  
    const viralYield \= viralResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
    expect(viralYield).to.be.greaterThan(normalYield \* 1.5); // Should be at least 50% higher  
      
    // Declining game activity  
    const decliningResults \= await simulator.runSimulation(\'normal\', \'declining\');  
    const decliningYield \= decliningResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
    expect(decliningYield).to.be.lessThan(normalYield \* 0.8); // Should be at least 20% lower  
  });  
    
  it("Should combine market and game activity impacts", async function() {  
    // Bull market \+ viral game (optimal scenario)  
    const bullViralResults \= await simulator.runSimulation(\'bull\', \'viral\');  
    const bullViralYield \= bullViralResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
      
    // Bear market \+ declining game (worst scenario)  
    const bearDecliningResults \= await simulator.runSimulation(\'bear\', \'declining\');  
    const bearDecliningYield \= bearDecliningResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
      
    // Extreme difference between best and worst scenarios  
    expect(bullViralYield / bearDecliningYield).to.be.greaterThan(3.0); // At least 3x difference  
  });  
});

### **4\. POC Deliverables (UPDATED)**

#### **4.1 Technical Documentation**

* Architecture diagrams showing component interaction between Saga Protocol chainlets and Mosaical platform  
* Smart contract specifications with function descriptions tailored for GameFi NFT lending  
* Saga Protocol integration documentation and chainlet compatibility checklist  
* GameFi yield simulation methodology and results by game category  
* Test coverage report with specific GameFi use cases  
* Cross-chainlet security considerations
* **Integration details for Alchemy NFT API**
* **Frontend integration approach using viem and wagmi**
* **Frontend framework migration details (Solid.js and Bootstrap)**

#### **4.2 Demo Environment (UPDATED)**

* Deployed contracts on Saga Protocol testnet  
* Basic CLI interface for interacting with Mosaical contracts  
* Simulation dashboard showing GameFi NFT yield projections  
* Monitoring tools for cross-chainlet GameFi asset pricing  
* Demo integration with at least 2 popular GameFi chainlets
* **Frontend demo using Solid.js and Bootstrap, integrated with smart contracts via viem/wagmi**

#### **4.3 Validation Report (UPDATED)**

* Success metrics evaluation against criteria  
* Performance analysis under various market and game activity conditions  
* Identified limitations and mitigation strategies  
* Recommendations for prototype phase  
* GameFi-specific risk assessment
* **Evaluation of Alchemy NFT API performance and reliability**
* **Assessment of viem/wagmi integration effectiveness**
* **Analysis of Solid.js/Bootstrap frontend performance and development experience**

### **5\. Success Metrics Validation Plan (UPDATED)**

#### **5.1 NFT Deposit/Withdrawal Testing**

**Test Cases:**

* Deposit/withdraw NFTs from multiple GameFi chainlets  
* Cross-collection NFT management  
* Attempt unauthorized withdrawals  
* Withdraw with outstanding loans
* **Test NFT data retrieval using Alchemy API before deposit/after withdrawal**

**Success Criteria:** 100% successful legitimate transactions, proper rejection of invalid operations, cross-chainlet compatibility, **accurate and timely NFT data retrieval via Alchemy API**

#### **5.2 GameFi Yield Collection Mechanism**

**Test Cases:**

* Simulate yield from multiple GameFi sources  
* Test yield distribution to loan repayment  
* Measure accuracy against expected yields by game type  
* Simulate in-game rewards as yield source

**Success Criteria:** Yield collection within 5% of expected values for established games, proper distribution to loan accounts, successful handling of multiple reward token types

#### **5.3 Oracle Price Feed Accuracy (UPDATED - Using Alchemy NFT API)**

**Test Cases:**

* Compare oracle prices (fetched via Alchemy) against actual GameFi marketplace data  
* Test pricing for NFTs with high vs. low in-game utility (metadata fetched via Alchemy)  
* Introduce price volatility scenarios  
* Simulate chainlet source failures
* **Test Alchemy API rate limits and error handling**

**Success Criteria:** Price accuracy within 8% of actual market values for GameFi NFTs, proper fallback behavior, appropriate utility-based valuation adjustments, **reliable data fetching from Alchemy API under various conditions**

#### **5.4 Market & Game Volatility Handling**

**Test Cases:**

* Simulate 40% price drops over 24 hours (common in GameFi)  
* Test rapid price oscillations during game events/updates  
* Simulate game shutdown/maintenance scenarios  
* Test changing user engagement levels

**Success Criteria:** System maintains stability during typical GameFi volatility, proper alerts generated, no critical failures, appropriate LTV adjustments based on game activity metrics

## **PROTOTYPE PHASE DETAILED BLUEPRINT (UPDATED)**

### **1\. Core Objectives (UPDATED)**

The Prototype will deliver a minimally interactive system with:

* Full user interface for GameFi NFT lending operations on Saga Protocol **built with Solid.js and Bootstrap**  
* Enhanced smart contracts with multi-chainlet and multi-collection support  
* DPO token implementation with basic trading functionality  
* Graduated liquidation thresholds based on GameFi utility metrics  
* Basic governance for protocol parameters  
* GameFi collection registry with chainlet integration
* **Frontend interaction with smart contracts using viem and wagmi**
* **Integration with Alchemy NFT API for comprehensive NFT data**

### **2\. Technical Architecture (UPDATED)**

#### **2.1 Smart Contract Upgrades**

// NFTVault V2 with enhanced GameFi support (No changes in core logic for Prototype)

contract NFTVaultV2 {  
    // Extended storage from V1  
    mapping(address \=\> mapping(address \=\> mapping(uint256 \=\> bool))) public deposits;  
    mapping(address \=\> bool) public supportedChainlets;  
    mapping(address \=\> mapping(address \=\> bool)) public supportedCollections;  
      
    // New storage for V2  
    // GameFi collection category mapping  
    mapping(address \=\> uint8) public gameCategory; // 1=RPG, 2=Racing, 3=Strategy, etc.  
      
    // GameFi NFT metadata cache  
    mapping(address \=\> mapping(uint256 \=\> bytes)) public nftMetadataCache;  
      
    // Batch deposit function for improved UX  
    function batchDepositNFTs(address collection, uint256\[\] calldata tokenIds) external {  
        require(isGameFiNFT(collection), "Not a supported GameFi NFT");  
          
        for (uint256 i \= 0; i \< tokenIds.length; i++) {  
            // Transfer NFT to vault  
            IERC721(collection).transferFrom(msg.sender, address(this), tokenIds\[i\]);  
            deposits\[msg.sender\]\[collection\]\[tokenIds\[i\]\] \= true;  
            emit NFTDeposited(msg.sender, collection, tokenIds\[i\]);  
              
            // Cache NFT metadata if not already cached  
            if (nftMetadataCache\[collection\]\[tokenIds\[i\]\].length \== 0\) {  
                bytes memory metadata \= fetchNFTMetadata(collection, tokenIds\[i\]);  
                nftMetadataCache\[collection\]\[tokenIds\[i\]\] \= metadata;  
            }  
        }  
    }  
      
    // Fetch NFT metadata (called internally)  
    function fetchNFTMetadata(address collection, uint256 tokenId) internal view returns (bytes memory) {  
        // For V2: Simple implementation to fetch tokenURI and return as bytes  
        // In V3, this will be enhanced to parse and store specific GameFi attributes  
        string memory tokenURI \= IERC721Metadata(collection).tokenURI(tokenId);  
        return bytes(tokenURI);  
    }  
      
    // Enhanced isGameFiNFT with GameFi category support  
    function isGameFiNFT(address collection) public view returns (bool) {  
        address chainlet \= getChainletFromCollection(collection);  
        return supportedChainlets\[chainlet\] &&   
               supportedCollections\[chainlet\]\[collection\] &&  
               gameCategory\[collection\] \> 0;  
    }  
      
    // Set GameFi category (admin/governance only)  
    function setGameCategory(address collection, uint8 category) external onlyAdminOrGovernance {  
        require(category \> 0 && category \<= 10, "Invalid game category");  
        gameCategory\[collection\] \= category;  
    }  
      
    // Additional functions from V1 with needed enhancements...  
}

contract LoanManagerV2 {  
    // Extended storage from V1  
    mapping(address \=\> mapping(address \=\> mapping(uint256 \=\> uint256))) public loans;  
    IPriceOracle public oracle;  
    mapping(address \=\> uint256) public gameFiUtilityMultiplier;  
      
    // New storage for V2  
    // Collection-specific interest rates (basis points)  
    mapping(address \=\> uint256) public collectionInterestRates;  
      
    // Health factors for loans (higher \= better, 1.0 \= liquidation threshold)  
    mapping(address \=\> mapping(address \=\> mapping(uint256 \=\> uint256))) public loanHealthFactors;  
      
    // Default interest rate in basis points (3% \= 300\)  
    uint256 public defaultInterestRate \= 300;  
      
    // Graduated liquidation thresholds  
    struct LiquidationThreshold {  
        uint256 healthFactor; // 1.0 \= 10000 (scaled by 10000\)  
        uint256 liquidationPct; // Percentage to liquidate at this threshold  
    }  
      
    // 3 levels of liquidation  
    LiquidationThreshold\[3\] public liquidationLevels;  
      
    constructor() {  
        // Initialize liquidation levels  
        liquidationLevels\[0\] \= LiquidationThreshold(9000, 25); // At 0.9 health, liquidate 25%  
        liquidationLevels\[1\] \= LiquidationThreshold(8000, 50); // At 0.8 health, liquidate 50%  
        liquidationLevels\[2\] \= LiquidationThreshold(7000, 100); // At 0.7 health, liquidate 100%  
    }  
      
    // Enhanced borrow with interest calculation  
    function borrow(address collection, uint256 tokenId, uint256 amount) external {  
        require(nftVault.deposits(msg.sender, collection, tokenId), "Not your NFT");  
          
        // Get NFT price from oracle  
        uint256 nftPrice \= oracle.getNFTPrice(collection, tokenId);  
          
        // Apply GameFi utility multiplier to base LTV  
        uint256 utilityFactor \= gameFiUtilityMultiplier\[collection\];  
        if (utilityFactor \== 0\) utilityFactor \= 100; // Default 100% (no adjustment)  
          
        // Calculate max LTV with utility adjustment  
        uint256 baseMaxLTV \= 50; // 50% base LTV  
        uint256 adjustedMaxLTV \= (baseMaxLTV \* utilityFactor) / 100;  
          
        // Cap at 70% max LTV for safety  
        if (adjustedMaxLTV \> 70\) adjustedMaxLTV \= 70;  
          
        uint256 maxLoan \= (nftPrice \* adjustedMaxLTV) / 100;  
        require(amount \<= maxLoan, "Exceeds LTV");  
          
        // Store loan amount  
        loans\[msg.sender\]\[collection\]\[tokenId\] \= amount;  
          
        // Initialize health factor (max \= 2.0, scales down as price decreases)  
        // Initial health \= NFT value / loan amount \* 0.5 (scaled by 10000\)  
        uint256 initialHealth \= (nftPrice \* 10000\) / amount / 2;  
        loanHealthFactors\[msg.sender\]\[collection\]\[tokenId\] \= initialHealth;  
          
        // Issue DPO tokens for the loan  
        dpoToken.mintDPOTokens(msg.sender, collection, tokenId, amount);  
          
        // Transfer loan amount to borrower  
        IERC20(lendingToken).transfer(msg.sender, amount);  
        emit LoanCreated(msg.sender, collection, tokenId, amount, initialHealth);  
    }  
      
    // Calculate interest for a loan  
    function calculateInterest(address user, address collection, uint256 tokenId, uint256 timestamp) public view returns (uint256) {  
        uint256 loanAmount \= loans\[user\]\[collection\]\[tokenId\];  
        if (loanAmount \== 0\) return 0;  
          
        // Get loan start time  
        uint256 loanStart \= loanStartTime\[user\]\[collection\]\[tokenId\];  
        uint256 timeElapsed \= timestamp \- loanStart;  
          
        // Get interest rate (collection-specific or default)  
        uint256 interestRate \= collectionInterestRates\[collection\];  
        if (interestRate \== 0\) interestRate \= defaultInterestRate;  
          
        // Calculate interest: principal \* rate \* time  
        // rate is in basis points (1% \= 100\)  
        // time is in seconds converted to years  
        return (loanAmount \* interestRate \* timeElapsed) / (10000 \* 365 days);  
    }  
      
    // Update loan health factor based on current NFT price  
    function updateHealthFactor(address user, address collection, uint256 tokenId) public returns (uint256) {  
        uint256 loanAmount \= loans\[user\]\[collection\]\[tokenId\];  
        if (loanAmount \== 0\) return 0;  
          
        // Get current NFT price  
        uint256 nftPrice \= oracle.getNFTPrice(collection, tokenId);  
          
        // Calculate health factor: NFT value / loan amount \* 0.5 (scaled by 10000\)  
        uint256 newHealth \= (nftPrice \* 10000\) / loanAmount / 2;  
        loanHealthFactors\[user\]\[collection\]\[tokenId\] \= newHealth;  
          
        return newHealth;  
    }  
      
    // Partial liquidation function  
    function liquidatePartial(address user, address collection, uint256 tokenId) external {  
        // Update health factor first  
        uint256 health \= updateHealthFactor(user, collection, tokenId);  
          
        // Determine liquidation level  
        uint256 liquidationPct \= 0;  
        for (uint256 i \= 0; i \< liquidationLevels.length; i++) {  
            if (health \<= liquidationLevels\[i\].healthFactor) {  
                liquidationPct \= liquidationLevels\[i\]\[i\];  
                break;  
            }  
        }  
          
        require(liquidationPct \> 0, "No liquidation needed");  
          
        uint256 loanAmount \= loans\[user\]\[collection\]\[tokenId\];  
        uint256 liquidationAmount \= (loanAmount \* liquidationPct) / 100;  
          
        // Execute partial liquidation via DPO token mechanism  
        dpoToken.executeLiquidation(user, collection, tokenId, liquidationAmount, liquidationPct);  
          
        // Reduce loan amount  
        loans\[user\]\[collection\]\[tokenId\] \-= liquidationAmount;  
          
        // If full liquidation, return remaining NFT value to user  
        if (liquidationPct \== 100\) {  
            nftVault.liquidationReturn(user, collection, tokenId);  
        }  
          
        emit LoanLiquidated(user, collection, tokenId, liquidationAmount, liquidationPct);  
    }  
      
    // Additional functions from V1 with needed enhancements...  
}

contract DPOTokenV2 is ERC20 {  
    // Ownership and admin contracts  
    address public admin;  
    address public loanManager;  
      
    // DPO token mappings  
    mapping(address \=\> mapping(uint256 \=\> uint256)) public nftTokenSupply;  
    mapping(address \=\> mapping(uint256 \=\> mapping(address \=\> uint256))) public tokenHoldings;  
      
    // NFT to DPO token mapping for lookup  
    mapping(address \=\> mapping(uint256 \=\> address)) public nftToDPOToken;  
      
    // DPO trading fee in basis points (0.5% \= 50\)  
    uint256 public tradingFee \= 50;  
    address public feeRecipient;  
      
    // Enhanced mint with amount-based supply  
    function mintDPOTokens(  
        address user,   
        address collection,   
        uint256 tokenId,   
        uint256 loanAmount  
    ) external onlyLoanManager {  
        // Calculate token supply based on loan amount  
        // 1 token per 0.001 ETH in loan value (configurable)  
        uint256 tokenSupply \= (loanAmount \* 1000\) / (10\*\*15);  
          
        \_mint(user, tokenSupply);  
        nftTokenSupply\[collection\]\[tokenId\] \= tokenSupply;  
        tokenHoldings\[collection\]\[tokenId\]\[user\] \= tokenSupply;  
          
        // Store reverse mapping for lookup  
        string memory tokenName \= string(abi.encodePacked("DPO-", IERC721Metadata(collection).symbol(), "-", tokenId.toString()));  
        ERC20 newDpoToken \= \_createChildToken(tokenName, tokenName);  
        nftToDPOToken\[collection\]\[tokenId\] \= address(newDpoToken);  
          
        emit DPOTokensMinted(user, collection, tokenId, tokenSupply);  
    }  
      
    // Trading functionality for DPO tokens  
    function tradeDPOTokens(  
        address collection,   
        uint256 tokenId,   
        address to,   
        uint256 amount  
    ) external {  
        require(tokenHoldings\[collection\]\[tokenId\]\[msg.sender\] \>= amount, "Insufficient DPO tokens");  
          
        // Calculate fee  
        uint256 fee \= (amount \* tradingFee) / 10000;  
        uint256 netAmount \= amount \- fee;  
          
        // Transfer tokens  
        tokenHoldings\[collection\]\[tokenId\]\[msg.sender\] \-= amount;  
        tokenHoldings\[collection\]\[tokenId\]\[to\] \+= netAmount;  
          
        // Transfer fee to fee recipient  
        if (fee \> 0\) {  
            tokenHoldings\[collection\]\[tokenId\]\[feeRecipient\] \+= fee;  
        }  
          
        // Reflect in ERC20 balances if using child tokens  
        address dpoTokenAddr \= nftToDPOToken\[collection\]\[tokenId\];  
        if (dpoTokenAddr \!= address(0)) {  
            ERC20 dpoToken \= ERC20(dpoTokenAddr);  
            bool success \= dpoToken.transferFrom(msg.sender, to, netAmount);  
            require(success, "Transfer failed");  
              
            if (fee \> 0\) {  
                success \= dpoToken.transferFrom(msg.sender, feeRecipient, fee);  
                require(success, "Fee transfer failed");  
            }  
        }  
          
        emit DPOTokensTraded(msg.sender, to, collection, tokenId, amount, fee);  
    }  
      
    // Execute liquidation by burning DPO tokens  
    function executeLiquidation(  
        address user,  
        address collection,  
        uint256 tokenId,  
        uint256 liquidationAmount,  
        uint256 liquidationPct  
    ) external onlyLoanManager {  
        uint256 totalSupply \= nftTokenSupply\[collection\]\[tokenId\];  
        uint256 burnAmount \= (totalSupply \* liquidationPct) / 100;  
          
        // Burn proportional tokens from all holders  
        address\[\] memory holders \= getTokenHolders(collection, tokenId);  
        for (uint256 i \= 0; i \< holders.length; i++) {  
            address holder \= holders\[i\];  
            uint256 holderBalance \= tokenHoldings\[collection\]\[tokenId\]\[holder\];  
              
            uint256 holderBurnAmount \= (holderBalance \* liquidationPct) / 100;  
            tokenHoldings\[collection\]\[tokenId\]\[holder\] \-= holderBurnAmount;  
              
            // Reflect in ERC20 balances if using child tokens  
            address dpoTokenAddr \= nftToDPOToken\[collection\]\[tokenId\];  
            if (dpoTokenAddr \!= address(0)) {  
                ERC20 dpoToken \= ERC20(dpoTokenAddr);  
                \_burn(holder, holderBurnAmount);  
            }  
              
            emit DPOTokensBurned(holder, collection, tokenId, holderBurnAmount);  
        }  
          
        // Reduce total supply  
        nftTokenSupply\[collection\]\[tokenId\] \= totalSupply \- burnAmount;  
    }  
      
    // Helper to get all token holders for an NFT  
    function getTokenHolders(address collection, uint256 tokenId) internal view returns (address\[\] memory) {  
        // Implementation would track holders in a mapping or array  
        // Simplified version for the blueprint  
        // In production: Use EnumerableMap or similar for gas-efficient holder tracking  
        return holderRegistry.getHolders(collection, tokenId);  
    }  
      
    // Create child DPO token (internal helper)  
    function \_createChildToken(string memory name, string memory symbol) internal returns (ERC20) {  
        // Implementation would deploy a child ERC20 contract  
        // Simplified version for the blueprint  
        return new ChildDPOToken(name, symbol, address(this));  
    }  
}

contract GameFiOracleV2 {  
    // Extended storage from V1  
    mapping(address \=\> uint256) public collectionFloorPrices;  
    mapping(address \=\> mapping(uint256 \=\> uint256)) public nftUtilityScores;  
      
    // New storage for V2  
    // Price history for volatility calculation  
    struct PricePoint {  
        uint256 timestamp;  
        uint256 price;  
    }  
      
    // Last 30 price points for each collection  
    mapping(address \=\> PricePoint\[30\]) public priceHistory;  
    mapping(address \=\> uint8) public historyIndex; // Current index in circular buffer  
      
    // Collection volatility (basis points, 1000 \= 10% daily volatility)  
    mapping(address \=\> uint256) public collectionVolatility;  
      
    // Multi-source aggregation weights (basis points, total should be 10000\)  
    struct PriceSource {  
        string name;  
        uint256 weight;  
    }  
      
    // Up to 5 price sources per collection  
    mapping(address \=\> PriceSource\[5\]) public priceSources;  
     
    // Function to update price from multiple sources (including Alchemy)
    function updatePrice(address collection) external onlyOracleMaintainer returns (uint256) {
        uint256 totalWeightedPrice = 0;
        uint256 totalWeight = 0;

        for (uint256 i = 0; i < priceSources[collection].length; i++) {
            PriceSource storage source = priceSources[collection][i];
            uint256 price = 0;

            // Fetch price based on source name
            if (keccak256(bytes(source.name)) == keccak256("Alchemy")) {
                // Call external function to fetch from Alchemy (requires off-chain worker or oracle network)
                price = IAlchemyOracle(alchemyOracleAddress).getFloorPrice(collection);
            } else if (keccak256(bytes(source.name)) == keccak256("ChainletAPI")) {
                 price = IChainletOracle(chainletOracleAddress).getFloorPrice(collection);
            } // Add other sources here

            if (price > 0) {
                totalWeightedPrice += price * source.weight;
                totalWeight += source.weight;
            }
        }

        require(totalWeight > 0, "No valid price sources");

        uint256 aggregatedPrice = totalWeightedPrice / totalWeight;

        // Store price history (circular buffer)
        uint8 currentIndex = historyIndex[collection];
        priceHistory[collection][currentIndex] = PricePoint({ timestamp: block.timestamp, price: aggregatedPrice });
        historyIndex[collection] = (currentIndex + 1) % 30;

        // Recalculate volatility (simplified for blueprint)
        collectionVolatility[collection] = calculateVolatility(collection);

        collectionFloorPrices[collection] = aggregatedPrice;
        return aggregatedPrice;
    }

    // Simplified volatility calculation (needs historical data)
    function calculateVolatility(address collection) internal view returns (uint256) {
        // In production, this would analyze priceHistory to calculate standard deviation or similar
        // For blueprint, return a mock value
        return 500; // 5% volatility mock
    }

    // Function to get NFT utility score (can still use cached metadata or fetch from chainlet)
    function getNFTUtilityScore(address collection, uint256 tokenId) public view returns (uint256) {
        // This can fetch from cached metadata or call a chainlet-specific utility oracle
        // For blueprint, assume a simple lookup
        return nftUtilityScores[collection][tokenId];
    }

    // Interface for external Alchemy Oracle caller
    interface IAlchemyOracle {
        function getFloorPrice(address collection) external view returns (uint256);
    }

     // Interface for external Chainlet Oracle caller
    interface IChainletOracle {
        function getFloorPrice(address collection) external view returns (uint256);
    }

    address public alchemyOracleAddress; // Address of the off-chain worker/oracle for Alchemy
    address public chainletOracleAddress; // Address of the chainlet-specific oracle

    // Admin function to set oracle addresses
    function setOracleAddresses(address _alchemyOracleAddress, address _chainletOracleAddress) external onlyAdmin {
        alchemyOracleAddress = _alchemyOracleAddress;
        chainletOracleAddress = _chainletOracleAddress;
    }

    // Admin function to add/update price sources
    function setPriceSource(address collection, uint8 index, string memory name, uint256 weight) external onlyAdmin {
        require(index < 5, "Index out of bounds");
        priceSources[collection][index] = PriceSource({ name: name, weight: weight });
    }

    // Admin function to set NFT utility score (for testing/initialization)
    function setNFTUtilityScore(address collection, uint256 tokenId, uint256 score) external onlyAdmin {
        nftUtilityScores[collection][tokenId] = score;
    }
}

#### **2.4 Frontend Implementation (UPDATED - Using Solid.js and Bootstrap)**

// Frontend Architecture using Solid.js and Bootstrap

This section outlines the frontend architecture and implementation details using Solid.js for reactive UI development and Bootstrap for styling and responsive design. The integration with smart contracts will be handled by viem and wagmi.

**Key Technologies:**

*   **Solid.js:** A declarative JavaScript library for creating user interfaces. Solid's fine-grained reactivity provides excellent performance by updating only the necessary parts of the DOM.
*   **Bootstrap:** A popular HTML, CSS, and JavaScript framework for developing responsive, mobile-first websites. It provides pre-built components and a grid system for rapid UI development.
*   **viem:** A low-level TypeScript interface for interacting with the Ethereum Virtual Machine (EVM). It provides a simple and efficient way to send transactions, read contract data, and listen for events.
*   **wagmi:** A collection of React Hooks (and core utilities) for interacting with Ethereum. While primarily known for React hooks, its core utilities and examples provide a strong foundation for integrating with EVM chains using viem in a framework-agnostic manner, which can be adapted for Solid.js.

**Architecture Overview:**

The frontend application will follow a component-based architecture, typical for modern JavaScript frameworks. Solid.js components will manage the UI state and rendering. Bootstrap will be used for styling and layout. viem and wagmi will handle all interactions with the smart contracts deployed on the Saga Protocol chainlet.

**Integration with Smart Contracts (using viem and wagmi):**

Interacting with smart contracts from the Solid.js frontend will involve using viem for low-level calls and transactions, and adapting wagmi's patterns for managing wallet connections, network switching, and contract interactions within Solid's reactivity model.

1.  **Wallet Connection:** Use wagmi's core utilities or a compatible library to handle wallet connection (e.g., WalletConnect, MetaMask). This will provide the necessary provider and signer for viem.
2.  **Contract Interaction:** Define contract ABIs and addresses. Use viem's `createPublicClient` and `createWalletClient` to interact with the blockchain. Read operations (e.g., getting loan status, fetching NFT details from the contract) will use `readContract`. Write operations (e.g., depositing NFT, borrowing, repaying) will use `writeContract` and handle transaction signing via the connected wallet.
3.  **State Management:** Solid.js's reactivity system (signals, memos, effects) will be used to manage the application state, including wallet connection status, fetched NFT data, loan information, and transaction states. Data fetched from smart contracts via viem will update Solid signals, triggering UI re-renders.
4.  **Event Handling:** Use viem's `watchContractEvent` or similar mechanisms to listen for smart contract events (e.g., `NFTDeposited`, `LoanCreated`). These events can trigger updates to the frontend state, providing real-time feedback to the user.

**Example (Conceptual) - Depositing an NFT:**

```javascript
import { createSignal, onCleanup } from 'solid-js';
import { useAccount, useWalletClient, usePublicClient } from 'wagmi'; // Assuming wagmi core utilities or a Solid adaptation
import { writeContract, parseAbi } from 'viem';

const nftVaultAbi = parseAbi([
  'function depositNFT(address collection, uint256 tokenId) external',
  // ... other functions
]);

const nftVaultAddress = '0x...'; // Deployed NFTVault contract address

function DepositNFTComponent(props) {
  const { address, isConnected } = useAccount(); // Get connected wallet address
  const { data: walletClient } = useWalletClient(); // Get wallet client for signing
  const publicClient = usePublicClient(); // Get public client for reading data

  const [isDepositing, setIsDepositing] = createSignal(false);
  const [error, setError] = createSignal(null);

  const handleDeposit = async () => {
    if (!isConnected || !walletClient) {
      setError("Please connect your wallet.");
      return;
    }

    setIsDepositing(true);
    setError(null);

    try {
      const { hash } = await writeContract({
        address: nftVaultAddress,
        abi: nftVaultAbi,
        functionName: 'depositNFT',
        args: [props.collectionAddress, props.tokenId],
        account: address,
        client: walletClient,
      });

      console.log("Transaction sent:", hash);

      // Optionally wait for transaction confirmation
      // const receipt = await publicClient.waitForTransactionReceipt({ hash });
      // console.log("Transaction confirmed:", receipt);

      setIsDepositing(false);
      // Trigger UI update or notification on success

    } catch (err) {
      console.error("Deposit failed:", err);
      setError(err.message);
      setIsDepositing(false);
    }
  };

  return (
    <div>
      <button onClick={handleDeposit} disabled={!isConnected || isDepositing()}>
        {isDepositing() ? "Depositing..." : "Deposit NFT"}
      </button>
      {error() && <p style={{ color: 'red' }}>Error: {error()}</p>}
    </div>
  );
}
```

**Styling (using Bootstrap):**

Bootstrap CSS classes and components will be used to style the application, ensuring a responsive and consistent look and feel across different devices. Custom CSS can be added to override or extend Bootstrap styles as needed.

```html
<!-- Example using Bootstrap classes -->
<div class="container mt-4">
  <div class="card">
    <div class="card-body">
      <h5 class="card-title">NFT Details</h5>
      <p>Collection: {{ nft.collection }}</p>
      <p>Token ID: {{ nft.tokenId }}</p>
      <button class="btn btn-primary">Deposit</button>
    </div>
  </div>
</div>
```

**Build Process:**

The project will use a build tool compatible with Solid.js (e.g., Vite, Parcel) to bundle the application code, including Solid.js components, viem/wagmi integration, and Bootstrap. The build process will compile Solid.js code, process CSS (including Bootstrap), and optimize assets for deployment.

#### **2.5 Database Considerations (UPDATED)**

// Database for off-chain data (No changes in core concept, still using PostgreSQL)

*   **PostgreSQL:** A powerful, open-source relational database system. It will be used to store off-chain data, including cached NFT metadata (fetched from Alchemy), historical price data, GameFi utility scores, and user-specific frontend preferences.

*   **Data Synchronization:** Mechanisms will be implemented to synchronize relevant on-chain data (e.g., deposit events, loan updates) with the off-chain database for faster querying and improved user experience. Data fetched from Alchemy API will also be stored and managed here.

*   **API Layer:** A backend API layer (e.g., using Node.js/Express or Python/Flask) will be developed to serve data from the PostgreSQL database to the Solid.js frontend and handle any necessary off-chain computations or integrations (e.g., calling Alchemy API from the backend if needed for rate limiting or security).

### **3\. Testing Framework (UPDATED)**

#### **3.1 Smart Contract Testing**

// Smart Contract Testing (No changes in core logic for Prototype)

#### **3.2 Oracle Testing (UPDATED - Focusing on Alchemy Integration)**

describe("GameFi Oracle Testing with Alchemy", function() {  
  let oracle;  
  const mockCollectionAddress \= "0xabcdef1234567890abcdef1234567890abcdef12";  
    
  beforeEach(async function() {  
    // Initialize oracle with mock API key and network  
    oracle \= new AlchemyNFTPriceOracle("mockApiKey", "eth-mainnet");  
      
    // Mock Alchemy API responses for testing  
    oracle.getFloorPrice \= async (collection) \=\> {  
      if (collection \=== mockCollectionAddress) {  
        return { floorPrice: { openSea: 100, looksRare: 102 } }; // Mock response structure  
      } else {  
        return null;  
      }  
    };  
      
    oracle.getNFTMetadata \= async (collection, tokenId) \=\> {  
        // Mock metadata response including attributes for utility score calculation
        return {
            metadata: {
                attributes: [
                    { trait_type: "level", value: 75 },
                    { trait_type: "rarity", value: "epic" },
                    { trait_type: "power", value: 850 }
                ]
            }
        };
    };

    // Mock utility calculation logic within the test or a helper function
    const calculateRpgUtilityScore = (attributes) => {
        let level = 0;
        let rarity = "common";
        let power = 0;

        for (const attr of attributes) {
            if (attr.trait_type === "level") level = attr.value;
            if (attr.trait_type === "rarity") rarity = attr.value;
            if (attr.trait_type === "power") power = attr.value;
        }

        const rarityBonus = {common: 0, uncommon: 5, rare: 15, epic: 30, legendary: 50}[rarity.toLowerCase()] || 0;
        return 100 + (level / 2) + rarityBonus + (power / 50);
    };

    // Override getNFTPrice to use mocked Alchemy calls and local utility calculation
    oracle.getNFTPrice = async (collection, tokenId) => {
        const floorData = await oracle.getFloorPrice(collection);
        const floorPrice = floorData ? floorData.floorPrice.openSea || floorData.floorPrice.looksRare || 0 : 0;

        const metadataResponse = await oracle.getNFTMetadata(collection, tokenId);
        const attributes = metadataResponse && metadataResponse.metadata && metadataResponse.metadata.attributes ? metadataResponse.metadata.attributes : [];
        const utilityScore = calculateRpgUtilityScore(attributes);

        if (utilityScore === 0) return floorPrice; // Default to floor if no score

        // Apply utility score as multiplier (baseline is 100)
        return (floorPrice * utilityScore) / 100;
    };

  });  
    
  it("Should fetch floor price from Alchemy", async function() {  
    const floorData \= await oracle.getFloorPrice(mockCollectionAddress);  
    expect(floorData).to.have.property("floorPrice");
    expect(floorData.floorPrice).to.have.property("openSea");
    expect(floorData.floorPrice.openSea).to.equal(100);
  });  
    
  it("Should fetch NFT metadata from Alchemy", async function() {
    const metadataResponse = await oracle.getNFTMetadata(mockCollectionAddress, 123);
    expect(metadataResponse).to.have.property("metadata");
    expect(metadataResponse.metadata).to.have.property("attributes");
    expect(metadataResponse.metadata.attributes).to.be.an("array");
  });

  it("Should calculate utility scores based on Alchemy metadata", async function() {  
    const utilityScore \= await oracle.getNFTPrice(  
      mockCollectionAddress,   
      123 // Token ID  
    );
      
    // Expected utility score calculation based on mocked metadata:
    // level: 75, rarity: "epic", power: 850
    // 100 + (75 / 2) + 30 + (850 / 50) = 100 + 37.5 + 30 + 17 = 184.5
    // Price calculation: floorPrice * utilityScore / 100 = 100 * 184.5 / 100 = 184.5
    expect(utilityScore).to.be.approximately(184.5, 0.1);  
  });  
});

#### **3.3 GameFi Yield Simulation Testing**

describe("GameFi Yield Simulation", function() {  
  let simulator;  
  const mockChainletId \= "0x1234567890123456789012345678901234567890";  
  const mockCollectionAddress \= "0xabcdef1234567890abcdef1234567890abcdef12";  
    
  beforeEach(async function() {  
    // Initialize simulator with test collections  
    simulator \= new GameFiYieldSimulator(  
      \[mockChainletId\],   
      {\[mockChainletId\]: \[mockCollectionAddress\]}  
    );  
      
    // Mock historical data fetching  
    simulator.fetchHistoricalYield \= async (chainlet, collection) \=\> {  
      if (collection \=== mockCollectionAddress) {  
        return Array(90).fill().map((\_, i) \=\> ({   
          day: i \+ 1,   
          yield: 0.002 \* (1 \+ 0.15 \* Math.sin(i / 8)) // 0.2% daily with sine wave variation  
        }));  
      } else {  
        return \[\];  
      }  
    };  
      
    // Mock utility impact analysis  
    simulator.fetchNFTSamplesWithUtility \= async () \=\> {  
      return Array(20).fill().map((\_, i) \=\> ({  
        tokenId: i \+ 1,  
        utilityScore: 100 \+ (i \* 5) // 100 to 195  
      }));  
    };  
      
    simulator.fetchNFTYieldHistory \= async (\_, \_\_, tokenId) \=\> {  
      const baseYield \= 0.002;  
      const utilityFactor \= tokenId % 20 ? (tokenId % 20) / 10 : 1; // 0.1 to 1.9  
      return Array(30).fill().map((\_, i) \=\> ({  
        day: i \+ 1,  
        yield: baseYield \* utilityFactor \* (1 \+ 0.1 \* Math.sin(i / 5))  
      }));  
    };  
      
    // Mock game engagement data  
    simulator.fetchGameEngagement \= async () \=\> ({  
      dailyActiveUsers: 15000,  
      averagePlaytime: 45, // minutes  
      retentionRate: 0.72,  
      monthlyRevenue: 320000  
    });  
      
    await simulator.initialize();  
  });  
    
  it("Should calculate accurate average yields", function() {  
    const key \= \`${mockChainletId}:${mockCollectionAddress}\`;  
    expect(simulator.dailyYields\[key\]).to.be.approximately(0.002, 0.0003);  
  });  
    
  it("Should calculate utility impact on yield", function() {  
    const key \= \`${mockChainletId}:${mockCollectionAddress}\`;  
    // Should be close to 1.0 if utility correctly predicts yield  
    expect(simulator.utilityImpact\[key\]).to.be.approximately(1.0, 0.2);  
  });  
    
  it("Should simulate different game activity scenarios", async function() {  
    // Normal game activity  
    const normalResults \= await simulator.runSimulation(\'normal\', \'normal\');  
    const normalYield \= normalResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
      
    // Viral game activity  
    const viralResults \= await simulator.runSimulation(\'normal\', \'viral\');  
    const viralYield \= viralResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
    expect(viralYield).to.be.greaterThan(normalYield \* 1.5); // Should be at least 50% higher  
      
    // Declining game activity  
    const decliningResults \= await simulator.runSimulation(\'normal\', \'declining\');  
    const decliningYield \= decliningResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
    expect(decliningYield).to.be.lessThan(normalYield \* 0.8); // Should be at least 20% lower  
  });  
    
  it("Should combine market and game activity impacts", async function() {  
    // Bull market \+ viral game (optimal scenario)  
    const bullViralResults \= await simulator.runSimulation(\'bull\', \'viral\');  
    const bullViralYield \= bullViralResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
      
    // Bear market \+ declining game (worst scenario)  
    const bearDecliningResults \= await simulator.runSimulation(\'bear\', \'declining\');  
    const bearDecliningYield \= bearDecliningResults\[mockChainletId\]\[mockCollectionAddress\].annualizedYield;  
      
    // Extreme difference between best and worst scenarios  
    expect(bullViralYield / bearDecliningYield).to.be.greaterThan(3.0); // At least 3x difference  
  });  
});

#### **3.4 Frontend Testing (NEW - Focusing on Solid.js, Bootstrap, viem/wagmi)**

This section outlines testing considerations for the updated frontend using Solid.js, Bootstrap, viem, and wagmi.

**Test Cases:**

*   **Wallet Connection:** Test connecting and disconnecting various wallet types (MetaMask, WalletConnect, etc.) using wagmi utilities.
*   **Network Switching:** Verify the application handles switching between different Saga Protocol chainlets correctly.
*   **NFT Display:** Test fetching and displaying user's NFTs using Alchemy API and Solid.js components.
*   **Contract Read Operations:** Test reading data from smart contracts (e.g., loan status, allowed collections) using viem and displaying it in the UI.
*   **Contract Write Operations:** Test sending transactions for depositing NFTs, borrowing, repaying, and withdrawing using viem and wagmi, ensuring proper transaction signing and confirmation handling.
*   **Error Handling:** Test how the frontend handles errors from wallet interactions, network issues, and smart contract calls.
*   **Responsive Design:** Verify the UI built with Solid.js and Bootstrap is responsive and functions correctly on different screen sizes.
*   **State Management:** Test that Solid.js's reactivity correctly updates the UI based on changes in application state and on-chain data.

**Success Criteria:** Successful wallet connection and network switching, accurate display of NFT and loan data, successful execution of smart contract transactions with proper feedback, graceful error handling, responsive UI, and efficient state updates.

### **4\. PROTOTYPE DELIVERABLES (UPDATED)**

#### **4.1 Technical Documentation (UPDATED)**

* Architecture diagrams showing component interaction between Saga Protocol chainlets and Mosaical platform  
* Smart contract specifications with function descriptions tailored for GameFi NFT lending  
* Saga Protocol integration documentation and chainlet compatibility checklist  
* GameFi yield simulation methodology and results by game category  
* Test coverage report with specific GameFi use cases  
* Cross-chainlet security considerations
* Integration details for Alchemy NFT API
* Frontend integration approach using viem and wagmi
* Frontend framework migration details (Solid.js and Bootstrap)
* **Detailed documentation on frontend architecture, component structure, and state management using Solid.js**
* **Guidelines and examples for using viem and wagmi for smart contract interactions in the Solid.js environment**
* **Documentation on Bootstrap customization and usage within the project**

#### **4.2 Demo Environment (UPDATED)**

* Deployed contracts on Saga Protocol testnet  
* Basic CLI interface for interacting with Mosaical contracts  
* Simulation dashboard showing GameFi NFT yield projections  
* Monitoring tools for cross-chainlet GameFi asset pricing  
* Demo integration with at least 2 popular GameFi chainlets
* Frontend demo using Solid.js and Bootstrap, integrated with smart contracts via viem/wagmi
* **Deployed frontend application accessible via a web browser**
* **Demonstration of key user flows: wallet connection, viewing owned NFTs (via Alchemy), depositing NFTs, borrowing against NFTs, repaying loans, and withdrawing NFTs.**

#### **4.3 Validation Report (UPDATED)**

* Success metrics evaluation against criteria  
* Performance analysis under various market and game activity conditions  
* Identified limitations and mitigation strategies  
* Recommendations for MVP phase  
* GameFi-specific risk assessment
* Evaluation of Alchemy NFT API performance and reliability
* Assessment of viem/wagmi integration effectiveness
* Analysis of Solid.js/Bootstrap frontend performance and development experience
* **User feedback on the Solid.js/Bootstrap frontend usability and performance**
* **Analysis of viem/wagmi transaction reliability and speed**

### **5\. Success Metrics Validation Plan (UPDATED)**

#### **5.1 NFT Deposit/Withdrawal Testing**

**Test Cases:**

* Deposit/withdraw NFTs from multiple GameFi chainlets  
* Cross-collection NFT management  
* Attempt unauthorized withdrawals  
* Withdraw with outstanding loans
* Test NFT data retrieval using Alchemy API before deposit/after withdrawal
* **Test deposit/withdrawal functionality via the Solid.js frontend using viem/wagmi**

**Success Criteria:** 100% successful legitimate transactions, proper rejection of invalid operations, cross-chainlet compatibility, accurate and timely NFT data retrieval via Alchemy API, **successful and reliable smart contract interactions initiated from the Solid.js frontend**

#### **5.2 GameFi Yield Collection Mechanism**

**Test Cases:**

* Simulate yield from multiple GameFi sources  
* Test yield distribution to loan repayment  
* Measure accuracy against expected yields by game type  
* Simulate in-game rewards as yield source

**Success Criteria:** Yield collection within 5% of expected values for established games, proper distribution to loan accounts, successful handling of multiple reward token types

#### **5.3 Oracle Price Feed Accuracy (UPDATED - Using Alchemy NFT API)**

**Test Cases:**

* Compare oracle prices (fetched via Alchemy) against actual GameFi marketplace data  
* Test pricing for NFTs with high vs. low in-game utility (metadata fetched via Alchemy)  
* Introduce price volatility scenarios  
* Simulate chainlet source failures
* Test Alchemy API rate limits and error handling

**Success Criteria:** Price accuracy within 8% of actual market values for GameFi NFTs, proper fallback behavior, appropriate utility-based valuation adjustments, reliable data fetching from Alchemy API under various conditions

#### **5.4 Market & Game Volatility Handling**

**Test Cases:**

* Simulate 40% price drops over 24 hours (common in GameFi)  
* Test rapid price oscillations during game events/updates  
* Simulate game shutdown/maintenance scenarios  
* Test changing user engagement levels

**Success Criteria:** System maintains stability during typical GameFi volatility, proper alerts generated, no critical failures, appropriate LTV adjustments based on game activity metrics

#### **5.5 Frontend Functionality Testing (NEW - Focusing on Solid.js, Bootstrap, viem/wagmi)**

**Test Cases:**

*   Test all UI components and their interactions in Solid.js.
*   Verify responsive design and layout using Bootstrap on various devices.
*   Test wallet connection and disconnection flows.
*   Test fetching and displaying user's NFT portfolio using Alchemy API calls integrated into Solid.js components.
*   Test initiating and confirming smart contract transactions (deposit, borrow, repay, withdraw) via the UI, using viem/wagmi.
*   Test real-time updates in the UI based on smart contract events.
*   Test form validation and user input handling in Solid.js.

**Success Criteria:** All UI components function as expected, responsive design is consistent, wallet operations are smooth, NFT data is displayed accurately, smart contract interactions are successful and provide clear feedback, UI updates react correctly to events, and user input is handled properly.

## **MVP PHASE DETAILED BLUEPRINT (UPDATED)**

### **1\. Core Objectives (UPDATED)**

The MVP will deliver a production-ready platform with:

* Full-featured user interface for GameFi NFT lending on Saga Protocol **built with Solid.js and Bootstrap**  
* Production-hardened smart contracts with comprehensive multi-chainlet and multi-collection support  
* Fully functional DPO token with trading and liquidity provision  
* Automated graduated liquidation based on real-time oracle data  
* On-chain governance for key protocol parameters  
* Comprehensive GameFi collection registry with automated chainlet integration and metadata parsing
* **Robust frontend interaction with smart contracts using viem and wagmi**
* **Production-grade integration with Alchemy NFT API for reliable and scalable NFT data**
* **Optimized Solid.js and Bootstrap frontend for performance and user experience**

### **2\. Technical Architecture (UPDATED)**

#### **2.1 Smart Contract Finalization**

// Finalized Smart Contract Architecture (No changes in core logic for MVP, focusing on audits and optimizations)

#### **2.2 Oracle Implementation (UPDATED - Production-ready Alchemy Integration)**

// Production-ready Alchemy NFT API Integration

This section details the production implementation of the Oracle, leveraging Alchemy NFT API for reliable and scalable NFT data. The Oracle will aggregate data from multiple sources, with Alchemy serving as a primary source for NFT metadata and floor prices.

**Key Components:**

*   **Off-chain Oracle Service:** A dedicated backend service responsible for fetching data from Alchemy NFT API and other sources. This service will handle API keys securely, manage rate limits, implement retry logic, and perform data validation.
*   **Data Aggregation and Validation:** The service will aggregate data from multiple sources (including Alchemy, chainlet-specific APIs, and potentially other marketplaces) and apply validation rules to ensure data accuracy and prevent manipulation. Weighted averages or other algorithms will be used to determine the final price and utility scores.
*   **On-chain Oracle Contract:** A smart contract on the Saga Protocol chainlet that receives validated data from the off-chain service. This contract will store the latest prices and utility scores, making them accessible to other smart contracts (e.g., LoanManager) via a secure mechanism (e.g., signed data feeds, Chainlink oracles).
*   **Alchemy NFT API Usage:** The off-chain service will utilize key Alchemy NFT API endpoints:
    *   `getNFTsForOwner`: To allow users to view their NFTs within the Mosaical platform.
    *   `getNFTMetadata`: To fetch detailed metadata for individual NFTs, including attributes used for utility score calculation.
    *   `getFloorPrice`: To retrieve collection floor prices from major marketplaces.
    *   `getContractMetadata`: To get details about NFT collections.

**Integration Flow:**

1.  The off-chain Oracle service periodically fetches data from Alchemy and other configured sources.
2.  Data is validated and aggregated to determine the final price and utility scores.
3.  The validated data is sent to the on-chain Oracle contract, typically signed by a trusted oracle operator.
4.  Smart contracts like LoanManager query the on-chain Oracle contract for the latest data when needed (e.g., during loan origination or liquidation checks).

#### **2.3 GameFi Yield Automation**

// GameFi Yield Automation (No changes in core concept for MVP, focusing on robustness and scalability)

#### **2.4 Frontend Implementation (UPDATED - Production-ready Solid.js and Bootstrap)**

// Production-ready Frontend using Solid.js and Bootstrap

This section details the production implementation of the frontend, built with Solid.js and Bootstrap, and integrated with smart contracts using viem and wagmi.

**Key Aspects:**

*   **Optimized Solid.js Components:** Components will be optimized for performance and reusability, leveraging Solid's fine-grained reactivity to minimize unnecessary re-renders.
*   **Comprehensive Bootstrap Usage:** Bootstrap will be used extensively for a consistent and responsive UI. Custom themes or components will be developed as needed while adhering to Bootstrap's principles.
*   **Robust viem/wagmi Integration:** The integration with smart contracts will be production-hardened, including comprehensive error handling, transaction status monitoring, and gas management. wagmi's core utilities will be used to provide a seamless wallet connection experience.
*   **State Management:** A well-defined state management strategy using Solid's primitives or a compatible library will ensure the application state is managed efficiently and predictably.
*   **Routing:** A Solid.js-compatible router will be implemented for navigation within the single-page application.
*   **Testing:** Comprehensive unit, integration, and end-to-end tests will be implemented for the frontend code.

**Example (Conceptual) - Viewing User's NFTs:**

```javascript
import { createResource } from 'solid-js';
import { useAccount } from 'wagmi'; // Assuming wagmi core utilities or a Solid adaptation
import { AlchemyNFTPriceOracle } from './alchemyService'; // Your service to interact with Alchemy

const alchemyOracle = new AlchemyNFTPriceOracle("YOUR_ALCHEMY_API_KEY", "YOUR_NETWORK");

function UserNFTs() {
  const { address, isConnected } = useAccount();

  const [nfts] = createResource(() => isConnected() ? address() : null, async (ownerAddress) => {
    const response = await alchemyOracle.getNFTsForOwner(ownerAddress);
    return response?.ownedNfts || [];
  });

  return (
    <div class="container mt-4">
      <h2>Your NFTs</h2>
      <Show when={isConnected()} fallback={<p>Please connect your wallet to view your NFTs.</p>}>
        <Show when={!nfts.loading} fallback={<p>Loading NFTs...</p>}>
          <div class="row">
            <For each={nfts()}>
              {(nft) => (
                <div class="col-md-4 mb-4">
                  <div class="card">
                    <img src={nft.media[0]?.gateway} class="card-img-top" alt={nft.title}/>
                    <div class="card-body">
                      <h5 class="card-title">{nft.title}</h5>
                      <p class="card-text">Collection: {nft.contract.name}</p>
                      <p class="card-text">Token ID: {nft.tokenId}</p>
                      {/* Add Deposit button or other actions */}
                    </div>
                  </div>
                </div>
              )}
            </For>
          </div>
        </Show>
      </Show>
    </div>
  );
}
```

#### **2.5 Database Finalization**

// Production-ready Database Implementation (No changes in core concept for MVP, focusing on scalability and security)

### **3\. Testing Framework (UPDATED)**

#### **3.1 Smart Contract Audits**

// Smart Contract Audits (No changes in core concept for MVP)

#### **3.2 Oracle Testing (UPDATED - Production Scenarios)**

describe("GameFi Oracle Production Testing with Alchemy", function() {  
  // Test cases for production scenarios, including:
  // - High-volume data fetching from Alchemy
  // - Handling Alchemy API errors and rate limits gracefully
  // - Data consistency and accuracy across multiple sources
  // - Latency and reliability of data updates on-chain
  // - Security of the off-chain oracle service
});

#### **3.3 GameFi Yield Simulation Testing**

// GameFi Yield Simulation Testing (No changes in core concept for MVP)

#### **3.4 Frontend Testing (UPDATED - Production Scenarios)**

describe("Frontend Production Testing (Solid.js, Bootstrap, viem/wagmi)", function() {  
  // Test cases for production scenarios, including:
  // - Performance testing of the Solid.js application
  // - Cross-browser and cross-device compatibility with Bootstrap
  // - Load testing of smart contract interactions via viem/wagmi
  // - Security testing of the frontend application
  // - User experience testing with a larger set of data and users
});

### **4\. MVP DELIVERABLES (UPDATED)**

#### **4.1 Production Codebase**

* Audited smart contracts deployed on Saga Protocol mainnet  
* Production-ready off-chain oracle service integrated with Alchemy NFT API and other sources  
* Production-ready backend API layer  
* Production-ready frontend application built with Solid.js and Bootstrap, integrated with smart contracts via viem/wagmi

#### **4.2 Deployment Infrastructure**

* Scalable and secure infrastructure for hosting backend services and the frontend application  
* Monitoring and alerting systems  
* CI/CD pipelines for automated testing and deployment

#### **4.3 Documentation and Support**

* Comprehensive technical documentation for all components  
* User documentation for the Mosaical platform  
* Support channels for users and developers

#### **4.4 Marketing and Launch**

* Marketing materials and launch strategy  
* Community building and engagement plan

## **References**

*   Alchemy NFT API Documentation: [1] https://docs.alchemy.com/reference/nft-api-quickstart
*   viem Documentation: [2] https://viem.sh/
*   wagmi Documentation: [3] https://wagmi.sh/react/guides/viem (Note: While primarily React-focused, core concepts and examples are relevant for viem integration)
*   Solid.js Documentation: [4] https://www.solidjs.com/docs/latest
*   Bootstrap Documentation: [5] https://getbootstrap.com/docs/5.3/getting-started/introduction/

---

**Author:** DevBernie
**Date:** June 18, 2025
