# CryptoZombies Complete GitHub Repository

## Repository Structure

```
crypto-zombies/
├── README.md
├── screenshots/
│   ├── lesson1-completion.png
│   ├── lesson2-gameplay.png
│   ├── lesson3-advanced-concepts.png
│   ├── lesson4-battle-system.png
│   ├── lesson5-erc721.png
│   └── lesson6-web3js.png
├── contracts/
│   ├── lesson1/
│   │   └── zombiefactory.sol
│   ├── lesson2/
│   │   ├── zombiefactory.sol
│   │   └── zombiefeeding.sol
│   ├── lesson3/
│   │   ├── zombiefactory.sol
│   │   ├── zombiefeeding.sol
│   │   └── zombiehelper.sol
│   ├── lesson4/
│   │   ├── zombiefactory.sol
│   │   ├── zombiefeeding.sol
│   │   ├── zombiehelper.sol
│   │   └── zombieattack.sol
│   ├── lesson5/
│   │   ├── zombiefactory.sol
│   │   ├── zombiefeeding.sol
│   │   ├── zombiehelper.sol
│   │   ├── zombieattack.sol
│   │   └── zombieownership.sol
│   └── lesson6/
│       ├── zombiefactory.sol
│       ├── zombiefeeding.sol
│       ├── zombiehelper.sol
│       ├── zombieattack.sol
│       └── zombieownership.sol
└── web-app/
    ├── index.html
    ├── css/
    │   └── style.css
    └── js/
        └── app.js
```

## Complete Code for All Lessons

### Lesson 1: Making the Zombie Factory (zombiefactory.sol)

```solidity
pragma solidity ^0.4.25;

contract ZombieFactory {

    event NewZombie(uint zombieId, string name, uint dna);

    uint dnaDigits = 16;
    uint dnaModulus = 10 ** dnaDigits;

    struct Zombie {
        string name;
        uint dna;
    }

    Zombie[] public zombies;

    mapping (uint => address) public zombieToOwner;
    mapping (address => uint) ownerZombieCount;

    function _createZombie(string _name, uint _dna) internal {
        uint id = zombies.push(Zombie(_name, _dna)) - 1;
        zombieToOwner[id] = msg.sender;
        ownerZombieCount[msg.sender]++;
        emit NewZombie(id, _name, _dna);
    }

    function _generateRandomDna(string _str) private view returns (uint) {
        uint rand = uint(keccak256(abi.encodePacked(_str)));
        return rand % dnaModulus;
    }

    function createRandomZombie(string _name) public {
        require(ownerZombieCount[msg.sender] == 0);
        uint randDna = _generateRandomDna(_name);
        _createZombie(_name, randDna);
    }
}
```

### Lesson 2: Zombies Attack Their Victims (zombiefeeding.sol)

```solidity
pragma solidity ^0.4.25;

import "./zombiefactory.sol";

contract KittyInterface {
  function getKitty(uint256 _id) external view returns (
    bool isGestating,
    bool isReady,
    uint256 cooldownIndex,
    uint256 nextActionAt,
    uint256 siringWithId,
    uint256 birthTime,
    uint256 matronId,
    uint256 sireId,
    uint256 generation,
    uint256 genes
  );
}

contract ZombieFeeding is ZombieFactory {

  KittyInterface kittyContract;
  
  modifier onlyOwnerOf(uint _zombieId) {
    require(msg.sender == zombieToOwner[_zombieId]);
    _;
  }

  function setKittyContractAddress(address _address) external {
    kittyContract = KittyInterface(_address);
  }

  function _triggerCooldown(Zombie storage _zombie) internal {
    _zombie.readyTime = uint32(now + cooldownTime);
  }

  function _isReady(Zombie storage _zombie) internal view returns (bool) {
    return (_zombie.readyTime <= now);
  }

  function feedAndMultiply(uint _zombieId, uint _targetDna, string _species) internal onlyOwnerOf(_zombieId) {
    Zombie storage myZombie = zombies[_zombieId];
    require(_isReady(myZombie));
    _targetDna = _targetDna % dnaModulus;
    uint newDna = (myZombie.dna + _targetDna) / 2;
    if (keccak256(abi.encodePacked(_species)) == keccak256(abi.encodePacked("kitty"))) {
      newDna = newDna - newDna % 100 + 99;
    }
    _createZombie("NoName", newDna);
    _triggerCooldown(myZombie);
  }

  function feedOnKitty(uint _zombieId, uint _kittyId) public {
    uint kittyDna;
    (,,,,,,,,,kittyDna) = kittyContract.getKitty(_kittyId);
    feedAndMultiply(_zombieId, kittyDna, "kitty");
  }
}
```

### Lesson 3: Advanced Solidity Concepts (zombiehelper.sol)

```solidity
pragma solidity ^0.4.25;

import "./zombiefeeding.sol";

contract ZombieHelper is ZombieFeeding {

  uint levelUpFee = 0.001 ether;

  modifier aboveLevel(uint _level, uint _zombieId) {
    require(zombies[_zombieId].level >= _level);
    _;
  }

  function withdraw() external onlyOwner {
    address _owner = owner();
    _owner.transfer(address(this).balance);
  }

  function setLevelUpFee(uint _fee) external onlyOwner {
    levelUpFee = _fee;
  }

  function levelUp(uint _zombieId) external payable {
    require(msg.value == levelUpFee);
    zombies[_zombieId].level++;
  }

  function changeName(uint _zombieId, string _newName) external aboveLevel(2, _zombieId) onlyOwnerOf(_zombieId) {
    zombies[_zombieId].name = _newName;
  }

  function changeDna(uint _zombieId, uint _newDna) external aboveLevel(20, _zombieId) onlyOwnerOf(_zombieId) {
    zombies[_zombieId].dna = _newDna;
  }

  function getZombiesByOwner(address _owner) external view returns(uint[]) {
    uint[] memory result = new uint[](ownerZombieCount[_owner]);
    uint counter = 0;
    for (uint i = 0; i < zombies.length; i++) {
      if (zombieToOwner[i] == _owner) {
        result[counter] = i;
        counter++;
      }
    }
    return result;
  }

}
```

### Lesson 4: Zombie Battle System (zombieattack.sol)

```solidity
pragma solidity ^0.4.25;

import "./zombiehelper.sol";

contract ZombieAttack is ZombieHelper {
  uint randNonce = 0;
  uint attackVictoryProbability = 70;

  function randMod(uint _modulus) internal returns(uint) {
    randNonce++;
    return uint(keccak256(abi.encodePacked(now, msg.sender, randNonce))) % _modulus;
  }

  function attack(uint _zombieId, uint _targetId) external onlyOwnerOf(_zombieId) {
    Zombie storage myZombie = zombies[_zombieId];
    Zombie storage enemyZombie = zombies[_targetId];
    uint rand = randMod(100);
    if (rand <= attackVictoryProbability) {
      myZombie.winCount++;
      myZombie.level++;
      enemyZombie.lossCount++;
      feedAndMultiply(_zombieId, enemyZombie.dna, "zombie");
    } else {
      myZombie.lossCount++;
      enemyZombie.winCount++;
      _triggerCooldown(myZombie);
    }
  }
}
```

### Lesson 5: ERC721 & Crypto-Collectibles (zombieownership.sol)

```solidity
pragma solidity ^0.4.25;

import "./zombieattack.sol";
import "./erc721.sol";
import "./safemath.sol";

contract ZombieOwnership is ZombieAttack, ERC721 {

  using SafeMath for uint256;

  mapping (uint => address) zombieApprovals;

  function balanceOf(address _owner) external view returns (uint256) {
    return ownerZombieCount[_owner];
  }

  function ownerOf(uint256 _tokenId) external view returns (address) {
    return zombieToOwner[_tokenId];
  }

  function _transfer(address _from, address _to, uint256 _tokenId) private {
    ownerZombieCount[_to] = ownerZombieCount[_to].add(1);
    ownerZombieCount[_from] = ownerZombieCount[_from].sub(1);
    zombieToOwner[_tokenId] = _to;
    emit Transfer(_from, _to, _tokenId);
  }

  function transferFrom(address _from, address _to, uint256 _tokenId) external payable {
    require (zombieToOwner[_tokenId] == msg.sender || zombieApprovals[_tokenId] == msg.sender);
    _transfer(_from, _to, _tokenId);
  }

  function approve(address _approved, uint256 _tokenId) external payable onlyOwnerOf(_tokenId) {
    zombieApprovals[_tokenId] = _approved;
    emit Approval(msg.sender, _approved, _tokenId);
  }
}
```

### ERC721 Interface (erc721.sol)

```solidity
pragma solidity ^0.4.25;

contract ERC721 {
  event Transfer(address indexed _from, address indexed _to, uint256 indexed _tokenId);
  event Approval(address indexed _owner, address indexed _approved, uint256 indexed _tokenId);

  function balanceOf(address _owner) external view returns (uint256);
  function ownerOf(uint256 _tokenId) external view returns (address);
  function transferFrom(address _from, address _to, uint256 _tokenId) external payable;
  function approve(address _approved, uint256 _tokenId) external payable;
}
```

### SafeMath Library (safemath.sol)

```solidity
pragma solidity ^0.4.25;

library SafeMath {

  function mul(uint256 a, uint256 b) internal pure returns (uint256) {
    if (a == 0) {
      return 0;
    }
    uint256 c = a * b;
    assert(c / a == b);
    return c;
  }

  function div(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a / b;
    return c;
  }

  function sub(uint256 a, uint256 b) internal pure returns (uint256) {
    assert(b <= a);
    return a - b;
  }

  function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    assert(c >= a);
    return c;
  }
}
```

### Lesson 6: App Front-ends & Web3.js

Create a simple web app interface with:

#### index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CryptoZombies Army</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <div class="container">
        <h1>CryptoZombies Army</h1>
        
        <div class="zombie-container">
            <!-- Zombies will appear here -->
            <div id="zombies"></div>
        </div>
        
        <div class="create-zombie">
            <h3>Create a New Zombie</h3>
            <input type="text" id="zombieName" placeholder="Enter zombie name">
            <button id="createZombieBtn">Create Zombie</button>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/web3@latest/dist/web3.min.js"></script>
    <script src="js/app.js"></script>
</body>
</html>
```

#### style.css
```css
body {
    font-family: 'Courier New', Courier, monospace;
    background-color: #2c3e50;
    color: #ecf0f1;
    margin: 0;
    padding: 0;
}

.container {
    max-width: 900px;
    margin: 0 auto;
    padding: 20px;
}

h1 {
    color: #1abc9c;
    text-align: center;
    margin-bottom: 40px;
}

.zombie-container {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 20px;
    margin-bottom: 40px;
}

.zombie-card {
    background-color: #34495e;
    border-radius: 10px;
    overflow: hidden;
    width: 200px;
    box-shadow: 0 4px 8px rgba(0,0,0,0.3);
    transition: transform 0.3s;
}

.zombie-card:hover {
    transform: translateY(-5px);
}

.zombie-card .zombie-name {
    background-color: #1abc9c;
    color: white;
    padding: 10px;
    font-weight: bold;
    text-align: center;
}

.zombie-card .zombie-stats {
    padding: 15px;
}

.zombie-card .zombie-dna {
    font-family: monospace;
    font-size: 0.8em;
    color: #95a5a6;
}

.create-zombie {
    background-color: #34495e;
    padding: 20px;
    border-radius: 10px;
    text-align: center;
}

input[type="text"] {
    padding: 10px;
    width: 70%;
    border: none;
    border-radius: 5px;
    margin-right: 10px;
}

button {
    background-color: #1abc9c;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 5px;
    cursor: pointer;
    font-weight: bold;
    transition: background-color 0.3s;
}

button:hover {
    background-color: #16a085;
}
```

#### app.js
```javascript
// Connect to Web3
window.addEventListener('load', async () => {
    // Modern browsers
    if (window.ethereum) {
        window.web3 = new Web3(ethereum);
        try {
            // Request account access
            await ethereum.request({ method: 'eth_requestAccounts' });
        } catch (error) {
            console.error("User denied account access");
        }
    }
    // Legacy browsers
    else if (window.web3) {
        window.web3 = new Web3(web3.currentProvider);
    }
    // Non-Ethereum browser
    else {
        alert('Please install MetaMask to use this dApp!');
    }
    
    // Initialize the app
    initApp();
});

function initApp() {
    // Contract ABI would go here (simplified for demonstration)
    const zombieFactoryABI = [
        // Methods from ZombieFactory contract
        {
            "constant": false,
            "inputs": [{"name": "_name", "type": "string"}],
            "name": "createRandomZombie",
            "outputs": [],
            "payable": false,
            "stateMutability": "nonpayable",
            "type": "function"
        },
        // More methods would be included here...
    ];
    
    // Contract address (this is just a placeholder)
    const zombieFactoryAddress = "0x123456789...";
    
    // Create contract instance
    const zombieFactoryContract = new web3.eth.Contract(zombieFactoryABI, zombieFactoryAddress);
    
    // Elements
    const createZombieBtn = document.getElementById('createZombieBtn');
    const zombieName = document.getElementById('zombieName');
    const zombiesDiv = document.getElementById('zombies');
    
    // Create zombie event
    createZombieBtn.addEventListener('click', async () => {
        if (zombieName.value.trim() === '') {
            alert('Please enter a zombie name!');
            return;
        }
        
        try {
            const accounts = await web3.eth.getAccounts();
            await zombieFactoryContract.methods.createRandomZombie(zombieName.value)
                .send({ from: accounts[0] });
                
            // This would normally be handled by an event, but for demo:
            alert(`Zombie "${zombieName.value}" created! (This is a demo - no actual transaction occurred)`);
            zombieName.value = '';
            
            // Add a dummy zombie to the UI for demonstration
            addZombieToUI({
                name: zombieName.value,
                dna: Math.floor(Math.random() * 10**16).toString(),
                level: 1,
                winCount: 0,
                lossCount: 0
            });
            
        } catch (error) {
            console.error("Error creating zombie:", error);
            alert("Error creating zombie. See console for details.");
        }
    });
    
    // Add zombie to UI function
    function addZombieToUI(zombie) {
        const zombieCard = document.createElement('div');
        zombieCard.className = 'zombie-card';
        
        zombieCard.innerHTML = `
            <div class="zombie-name">${zombie.name}</div>
            <div class="zombie-stats">
                <p><strong>Level:</strong> ${zombie.level}</p>
                <p><strong>Wins:</strong> ${zombie.winCount} / <strong>Losses:</strong> ${zombie.lossCount}</p>
                <p class="zombie-dna">DNA: ${zombie.dna}</p>
            </div>
        `;
        
        zombiesDiv.appendChild(zombieCard);
    }
    
    // For demonstration, add some sample zombies
    addZombieToUI({
        name: "Brainiac",
        dna: "1234567890123456",
        level: 2,
        winCount: 3,
        lossCount: 1
    });
    
    addZombieToUI({
        name: "DeadWalker",
        dna: "6543210987654321",
        level: 1,
        winCount: 0,
        lossCount: 0
    });
}
```
