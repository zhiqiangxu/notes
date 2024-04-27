# Introduction

Offchain Labs published a [security disclosure](https://medium.com/offchainlabs/security-disclosure-289a4ad50709) on OP Labs' fault dispute game module.

The findings are vavlid and critical, but the description is not so clear to many.

So let's dive into the details.

# Principle

OP Labs' (permissionless) fault dispute game before the recent fix works like this:
1. Anyone can publish a claim.
2. After the counter party's time credit(3.5 days) is used up, the claim can be resolved. 
   1. At that time, it's considered valid if:
      1. It's not countered. (**I**)
      2. All disputes for this claim are countered themselves. (**II**)
   2. Invalid otherwise.


Keen readers may have already discovered the problem here:

The claim can be resolved **immediately** as long as the counter party's time credit is used up, even if the claimant has plenty of time left. So if an evil challenger counters a claim at the last second of his time credit, and after 1 second, he can resolve the claim, and what's worse, the claim is considered invalid since it satisfies neither rule **I** nor **II** above.

So the direct fix from OP Labs is to introduce a `CLOCK_EXTENSION` concept, if the remaining time for the moving party is less than `CLOCK_EXTENSION`, the remaining time will be lifted to `CLOCK_EXTENSION` or `2*CLOCK_EXTENSION`, giving the other party enough time to respond! ([source](https://github.com/ethereum-optimism/optimism/blob/826a7bdced9d14bbb121aa73161c2108b9097677/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L355-L360))

The side effect is that the game may last longer than 7 days, since the remaining time will always be `CLOCK_EXTENSION` or `2*CLOCK_EXTENSION` if both parties move fast enough. But the safety is still guaranteed by the final one step.