# Parity WhitelistedMultisig contract bug bounty

  * https://www.reddit.com/r/ethdev/comments/6qaxc1/bug_bounty_on_the_consensys_and_whitelisted/
  * Date: 2017-08-31
  * Reward: 3 ETH

## Contracts

[WhitelistedMultisig.sol](./contracts/WhitelistedMultisig.sol)

## Report

I've discovered an issue in the Whitelisted Multisig wallet code that allows for something I would call a hidden owner attack.
The issue is contained to the "multiowned" contract and is caused by missing address duplicity check (necessity of which is implied in the addOwner function) in the constructor code. First I will describe the attack and then delve into how it works.
The attack has three steps:

1) First we deploy the contract with an array of owners with one address duplicated, something like this:
["0xca35b7d915458ef540ade6068dfe2f44e8fa733c", "0x14723a09acff6d2a60dcdf7aa4aff308fddc160c", "0x14723a09acff6d2a60dcdf7aa4aff308fddc160c"]

2) Then we call removeOwner with the duplicated address as an argument: removeOwner("0x14723a09acff6d2a60dcdf7aa4aff308fddc160c") After this call, isOwner("0x14723a09acff6d2a60dcdf7aa4aff308fddc160c") correctly returns false.

3) Now we remove an owner preceding the duplication:
removeOwner("0xca35b7d915458ef540ade6068dfe2f44e8fa733c") isOwner("0xca35b7d915458ef540ade6068dfe2f44e8fa733c") now returns false, but isOwner("0x14723a09acff6d2a60dcdf7aa4aff308fddc160c") suddenly returns true again!

How did this happen?

When we called removeOwner with the duplicated address, it was correctly removed from m_ownerIndex array, but the second instance remained in the m_owners array. At this point, there is no entry in the m_ownerIndex array pointing to the index of the duplicated address, so isOwner returns false, but that changes when we remove the second address. After removal, reorganizeOwners function is called and the orginal place of the address being removed is taken by the dormant instance of the duplicated address.
Level of exploitability depends on the implementation of the UI for the wallet, because on closer examination of the contract's state, it is possible to detect something is not quite right after step b), but anybody relying on isOwner function, or m_ownerIndex array to check the ownership will be completely fooled.
One scenario in which this can be exploited is when the wallet is being transferred between owners, it is possible to convince user or users that they are sole owners of the wallet, when that's not the case.
Easiest fix is adding owner duplicity check to the constructor.

```javascript
    // constructor is given number of sigs required to do protected "onlymanyowners" transactions
    // as well as the selection of addresses capable of confirming them.
    function multiowned(address[] _owners, uint _required) {
        require(_required > 0);
        require(_owners.length >= _required);
        m_numOwners = _owners.length;
        for (uint i = 0; i < _owners.length; ++i) {
            m_owners[1 + i] = uint(_owners[i]);
            m_ownerIndex[uint(_owners[i])] = 1 + i;
        }
        m_required = _required;
    }
```

```javascript
    function reorganizeOwners() private {
        uint free = 1;
        while (free < m_numOwners)
        {
            while (free < m_numOwners && m_owners[free] != 0) free++;
            while (m_numOwners > 1 && m_owners[m_numOwners] == 0) m_numOwners--;
            if (free < m_numOwners && m_owners[m_numOwners] != 0 && m_owners[free] == 0)
            {
                m_owners[free] = m_owners[m_numOwners];
                m_ownerIndex[m_owners[free]] = free;
                m_owners[m_numOwners] = 0;
            }
        }
    }
```