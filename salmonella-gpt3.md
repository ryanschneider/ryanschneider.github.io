**On Salmonella and GPT-3**

# Introduction

Recently I've been thinking about MEV and more generally EVM introspection.  I think it's self evident that the majority of Ethereum users do not really "do the right thing" when sending a transaction.

The "right thing" would be something like:

- Find the contract source (e.g on etherscan)
- Confirm the source you have matches the actual deployed smart contract.
- Read the whole source of the contract
- Simulate (either via tooling or mentally) what actually happens when you submit your contract
- Confirm that the outputs match your expectations
- Furthermore, keep in mind that contracts can be _designed_ to be hard to simulate correctly.

The last point is important: you should by default assume every contract is out to get you, not that everyone is behaving honestly.

Then of course there's the "Dark Forest" and MEV, and that even if you do all the above, you might still get sandwiched or otherwise attacked by other actors in the mempool, so really the next step isn't to call `eth_sendRawTransaction` but is instead to get your transaction to a miner/block builder through some other out-of-protocol means (e.g. FlashBots).

# Enter GPT-3

Anyways, while thinking about all this, I also recently saw Simon Wilson's post on using GPT-3 to understand code (https://simonwillison.net/2022/Jul/9/gpt-3-explain-code/).

That got me thinking: could GPT-3 be used to do some of those early steps?  I thought for sure the answer would be "no", but let's find out!

Anyways, Simon links to another article where he explains how to sign-up for the OpenAI beta and to use the GPT-3 playground.  I always assumed access to GPT-3 was much harder than it was, I was actually up and running in just a couple minutes.

Anyways, the first thing I did was:

- Download the WETH contract code from etherscan.
- Paste it into the playground (after removing the long trailing license comment)
- And added the prompt "What is this code?"

I was actually pretty surprised by it's response!

> The code above is a Solidity contract that allows users to deposit, withdraw, and transfer Ether (ETH). The contract is called "WETH9" and is a version of the popular "Wrapped Ether" (WETH) contract.
> 
> The contract has a number of features that make it useful for handling ETH:
> 
> The contract has a name, symbol, and decimals. This allows the contract to be used like a token on the Ethereum network.
> 
> The contract has a balanceOf mapping that tracks the balance of each user.
> 
> The contract has an allowance mapping that tracks how much ETH each user has allowed another user to spend on their behalf.
> 
> The contract has a deposit function that allows users to deposit ETH into the contract.
> 
> The contract has a withdraw function that allows users to withdraw ETH from the contract.
> 
> The contract has a transfer function that allows users to transfer ETH to another user.
> 
> The contract has a transferFrom function that allows users to transfer ETH from one user to another. This function is used by the Ethereum network to transfer ETH from one user to another.

Pretty good!  It seems to conflate WETH and ETH a bit too often than it should, but otherwise seems to get the gist of what's going on.

# Can I trick GPT-3?

So ya, it does a pretty good job at explaining the code.  But does it really _understand_ what it just said?  Well, let's test that.  One way to do that, would be to "tweak" part of the standard contract, and see if GPT-3 can catch that.

Since I've been thinking about MEV, and the cat and mouse around MEV, I'd also been recently thinking about the "Salmonella attack", a clever "let's MEV the MEV-ers" attack explained here: https://github.com/Defi-Cartel/salmonella

The TLDR is:

- MEV searchers use sandwich attacks to surround innocent transactions with buy and sell orders.
- These searchers usually find targets by simulating transactions they find in the public mempool
- One can potentially use this their own advantage and in turn extract MEV from the searchers

I don't think the full code of the Salmonella contract is available, but the above article does show the main difference in the transfer function.  For comparison, here's the standard WETH `transfer` function:

```solidity
contract WETH9 {
    // ....

    function transfer(address dst, uint wad) public returns (bool) {
        return transferFrom(msg.sender, dst, wad);
    }

    function transferFrom(address src, address dst, uint wad)
        public
        returns (bool)
    {
        require(balanceOf[src] >= wad);

        if (src != msg.sender && allowance[src][msg.sender] != uint(-1)) {
            require(allowance[src][msg.sender] >= wad);
            allowance[src][msg.sender] -= wad;
        }

        balanceOf[src] -= wad;
        balanceOf[dst] += wad;

        Transfer(src, dst, wad);

        return true;
    }

    // ...
}
```

While here's the Salmonella-fied version of WETH, borrowing the `_transfer` function from the above link.

```solidity
contract WETH9 {
    // ....

    // let's use the public WETH address for these for simplicity
    address ownerA = 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2;
    address ownerB = 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2;

    function transfer(address dst, uint wad) public returns (bool) {
        return _transfer(msg.sender, dst, wad);
    }

    function _transfer(address sender, address recipient, uint256 amount) internal virtual {
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");
        uint256 senderBalance = _balances[sender];
        require(senderBalance >= amount, "ERC20: transfer amount exceeds balance");
        if (sender == ownerA || sender == ownerB) {
            _balances[sender] = senderBalance - amount;
            _balances[recipient] += amount;
        } else {
            _balances[sender] = senderBalance - amount;
            uint256 trapAmount = (amount * 10) / 100;
            _balances[recipient] += trapAmount;
        }

       emit Transfer(sender, recipient, amount);
    }

// ...
}   
```

The main difference, of course, is that while it still _emits_ a "full" Transfer event, it actually only tranfers 10% of the amount to the recipient balance (for any senders other than the contract owner).  This "tweak" was enough to trick some MEV searchers, will GPT-3 be able to catch the tweak?  

With this change in place to the code, I added the prompt "What is different about this transfer function and the usual one?", and was totally blown away by what came back!

> What is different about this transfer function and the usual one?
> 
> 
> It includes a check to see if the sender is either ownerA or ownerB. If they are, then the amount is transferred as normal. If they are not, then only a portion of the amount is transferred.

Ok, wow, that's pretty good. Really impressive in fact.  But does it really _understand that_?

# Implications

This all seems _somewhat_ promising.  Promising enough to play around with more.  I could think of two immediate scenarios that could be further explored.  

However, in both scenarios, GPT-3 needs to better understand what's going on to be trusted.  Let's see if it does.

## Etherscan

The first one is Etherscan, since that's where most people already go to explore EVM code.  What if etherscan offered an easy way to interact with a GPT-3 instance already "prompted" with the contract code?  The prompt could even be preseeded with common questions like "What does this contract do?".

This seems like a cool feature, and isn't something I can really "mock up" in a meaningful way, so I'm skipping it for now.

## Metamask

IMO this one is even more interesting, and goes back to the beginning of this spiel where I said that users should really understand what their transactions are doing before sending them.  Can GPT-3 help with that?

Imagine that MetaMask used GPT-3 to "simulate" a transaction and prompted for the results.  Would the response be something worth sharing with an end user?

To mock that up, let's try this prompt, importantly _still using the modified version_ of WETH where the transfer function behaves non-standard:

> Given that 0x4bE0Cd2553356b4ABb8b6a1882325dAbC8d3013D and 0x1B57edAb586cBDABD4d914869Ae8bB78dbC05571 both start with 60000 WETH , what happens if address 0x4bE0Cd2553356b4ABb8b6a1882325dAbC8d3013D calls transfer(0x1B57edAb586cBDABD4d914869Ae8bB78dbC05571, 40000)?  Which events are emitted, and what is the final balance of 0x4bE0Cd2553356b4ABb8b6a1882325dAbC8d3013D and 0x1B57edAb586cBDABD4d914869Ae8bB78dbC05571?

We get back this response:

> The events emitted are:
> 
> Transfer(0x4bE0Cd2553356b4ABb8b6a1882325dAbC8d3013D, 0x1B57edAb586cBDABD4d914869Ae8bB78dbC05571, 40000)
> 
> The final balance of 0x4bE0Cd2553356b4ABb8b6a1882325dAbC8d3013D is 20000 WETH and the final balance of 0x1B57edAb586cBDABD4d914869Ae8bB78dbC05571 is 100000 WETH.

Hmm, ok.  Maybe it doesn't really understand what's going on here afterall (the final balances should be 20000 and 24000 since only 10% of WETH is transferred, the rest is lost).

Can it get there with more prompting?

I added `Are any WETH lost in the process?`.  To which I got:

> address 0x4bE0Cd2553356b4ABb8b6a1882325dAbC8d3013D starts with 60000 WETH
> address 0x1B57edAb586cBDABD4d914869Ae8bB78dbC05571 starts with 60000 WETH
> 
> 
> The final balance of 0x4bE0Cd2553356b4ABb8b6a1882325dAbC8d3013D is 20000 WETH
> The final balance of 0x1B57edAb586cBDABD4d914869Ae8bB78dbC05571 is 100000 WETH
> 
> No WETH are lost in the process.

# So, does it _really_ understand?

Obviously it doesn't, but sometimes _feels_ tantilizingly close.

Furthermore, it seems _very_ fragile.  I'll admit, the first couple times I tried this, I accidentally copy/pasted wrong and deleted the `transfer` function, leaving only Salmonella's `_transfer`.  This confused GPT-3, which responded with `The only difference is that this transfer function has an extra requirement that the sender's balance must be greater than or equal to the value being transferred.`.  Wait what?

After correcting that, I then realized I forgot to define `ownerA` and `ownerB`, which lead to this rather surprsing response:

> What is different about this transfer function and the usual one?
> 
> The usual transfer function looks like this:
> 
> function transfer(address dst, uint wad) public returns (bool) {
>     require(balanceOf[msg.sender] >= wad);
>     balanceOf[msg.sender] -= wad;
>     balanceOf[dst] += wad;
>     Transfer(msg.sender, dst, wad);
>     return true;
> }
> 
> However, in this contract, the transfer function has been modified to send a percentage of the amount being transferred to a different address. In this case, the percentage is 10%, and the address that the extra amount is sent to is ownerA or ownerB.

Nice try, but that's not what's actually going on (the extra amount simply disappears, it doesn't end up in anyone's balance).

So, a couple observations there:

- GPT-3 seems to _try_ to tell you what code does even if the code doesn't work.
- The responses vary wildly in correctness.
