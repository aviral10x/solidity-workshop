# Safe transfer of contract ownership

**Warning: the code in this article is only used as examples - it has not been unit tested or undergone any kind of serious review**

Having a master account, or "owner", of a Solidity contract is a common way of ensuring that some of its functions are accessible only to a specific account. Here is a simple and well-used template for an owner contract:

```
contract Owned {

  address public owner;
  
  modifier onlyOwner() {
    require(msg.sender == owner);
    _;
  }
  
  constructor() public {
    owner = msg.sender;
  }

  function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != 0);
    owner = newOwner;
  }
  
}
```

The contract above will allow you to:

1. Set an `owner`.
2. Block functions from being called by anyone who is not the `owner` (through the `onlyOwner` modifier).
3. Transfer ownership to another account.

Additionally, the transfer function can only be called by the current owner which means that nobody but the current owner can change the owner address. 

This is very good, but the transfer function still makes this contract unsafe. First of all, if you look at the constructor, you see that the account that uploads the contract automatically becomes the owner which guarantees that the owner is set to an "active account", i.e. an account that must have made at least one transaction. That is not the case for the `transferOwnership` function, because even though it cannot be set to 0 it could still be given the address of an account for which nobody has the private key. This could happen by accident, like if the current owner happens to type in the wrong address, or if something goes wrong when the function arguments are packed by the caller's contract API. Additionally, if the owner address is set to a bad address it will be impossible to rectify because the owner is the only account allowed to change it back.

There are several ways to add safeguards against this type of error; one is to require that the new owner makes a transaction to the contract before the transfer actually happens. Example:

```
contract Owned {

  address public owner;
  address public ownerTransf; // temporary storage

  modifier onlyOwner() {
    require(msg.sender == owner);
    _;
  }

  constructor() public {
    owner = msg.sender;
  }

  function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != 0);
    // Instead of swapping owner, we store the 'newOwner' address in the 'ownerTransf' variable.
    ownerTransf = newOwner;
  }
  
  // New functions
  
  // Cancel any on-going transfer.
  function cancelOwnershipTransfer() public onlyOwner {
    ownerTransf = 0;
  }
  
  // Lets the new owner claim ownership.
  function claimOwnership() public {
    require(msg.sender == ownerTransf);
    owner = ownerTransf;
    ownerTransf = 0;
  }
  
}
```

Now we get the same safety that we get from using `msg.sender` - both in the constructor and the transfer process. It does come with some additional code and gas costs, but that is a low price to pay for security; just keep in mind that while this is good, it does not make the ownership process entirely safe, for example the owner could still end up being a contract account in which there is no code for actually managing the contract. Another thing that could happen is that the ownership is transferred to an active external account but the holder loses their access to it, for example by losing the private key.

Another way to make sure that a contract will continue to function is by redundancy, i.e. to have more then one owner account. To give an example of this, here is a simple multi-owner contract built on the same principle as the single-owner one:

```
contract OwnedByTwo {

  address[2] public owners;
  address[2] public ownerTransfs; // temporary storage
  
  modifier onlyOwner() {
    require(msg.sender == owners[0] || msg.sender == owners[1]);
    _;
  }  
  
  constructor() public {
    owners[0] = msg.sender;
    owners[1] = msg.sender;
  }

  function transferOwnership(address newOwner, uint ownerIndex) public onlyOwner {
    require(ownerIndex < 2);
    require(newOwner != 0);
    // Instead of swapping owner, we store the 'newOwner' address in the 'ownerTransf' variable.
    ownerTransfs[ownerIndex] = newOwner;
  }
  
  // Cancel any on-going transfer.
  function cancelOwnershipTransfer(uint ownerIndex) public onlyOwner {
    require(ownerIndex < 2);
    ownerTransfs[ownerIndex] = 0;
  }
  
  // Lets the new owner claim ownership.
  function claimOwnership(uint ownerIndex) public {
    require(ownerIndex < 2);
    require(msg.sender == ownerTransfs[ownerIndex]);
    owners[ownerIndex] = ownerTransfs[ownerIndex];
    ownerTransfs[ownerIndex] = 0;
  }
  
}
```

In this case, there are two accounts that have the same level of access to the contract's functionality. Note that the claim function is almost redundant here, because the owner could just transfer the second owner account without ever touching his own, so even if he messes up it could still be possible to revert, but he could still cause problems, for example by providing the transfer function with the wrong `ownerIndex`. All things considered, it could still be good to have a two-step process to ensure that the transfer is done right.

To summarize - You should always make sure that there is a sound process in place to manage the ownership of a contract, because not doing so could have disasterous consequences. This article presents a few different ways in which this could be done.
