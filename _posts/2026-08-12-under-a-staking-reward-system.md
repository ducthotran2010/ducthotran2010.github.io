---
layout: post
title: "Under a staking reward system"
description: >-
  A staking reward has to be right at every moment, nobody is running a job to keep it that way, and
  changing your stake must not force a payout. The accounting that does all three.
tags: [web3]
---

<p class="runbox"><b>Versions.</b> Source links are pinned: <code>ronin-chain/dpos-contract</code> at
<b>348e2e36</b>. The runs below are <code>forge</code> against that commit, <code>solc 0.8.25</code>,
EVM <code>istanbul</code>. Every contract quoted here is upstream's at that commit. The tests are
mine.</p>

**TL;DR** A pool owes a different number to every person in it, and the obvious way to get those
numbers is to loop over them. I wrote one of these, and it has no loop. Nothing in it runs on a
timer, and it has never read the list of people it pays. The loop dies at about 1,274 stakers and
there is no timer to run, so what replaces both is one stored number for the pool and four for each
staker. Your figure is then right at every moment, and you can move your stake without being paid
out against your will.

## Four rounds of a pool paying out

Open a staking page and look at your pending reward. It is higher than when you last opened it, you
sent no transaction to make that happen, and nothing ran on your behalf while you were away.

Somebody still has to do that arithmetic. Earnings arrive for a whole pool at once, in batches, and
each batch has to become a different number for every person in the pool, in proportion to what they
held and for how long.

I wrote that part for a staking system. Here is what it has to produce, over four rounds of one pool
earning a hundred tokens in each. Alice opens the pool alone, Bob matches her, neither of them moves
for a round, then Alice adds two hundred more. Money added to a stake counts from the round after it
arrives, so every round below is priced on stakes that stood for the whole of it.

<figure class="diagram">
<svg viewBox="0 0 620 196" xmlns="http://www.w3.org/2000/svg" role="img"
     aria-label="Four rows, one per round. Each row draws the pool as a bar to scale, split into the part alice holds and the part bob holds, and the two numbers on the right are what that round paid them. Alice holds the pool alone in round one and takes all hundred tokens, they hold half each in rounds two and three and take fifty each, and in round four alice holds three hundred of four hundred and takes seventy five. After four rounds alice has two hundred and seventy five and bob has one hundred and twenty five.">
  <style>.rd-l{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.rd-h{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.rd-hc{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:end}.rd-in{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.rd-n{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75;text-anchor:end}.rd-tot{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b;text-anchor:end}.rd-a{fill:#f1efea;stroke:#b8b4ab;stroke-width:1.2}.rd-b{fill:#e3e0d8;stroke:#b8b4ab;stroke-width:1.2}.rd-rule{stroke:#b8b4ab;stroke-width:1;fill:none}</style>
  <text class="rd-h" x="8" y="24">the pool earns 100 tokens in every round</text>
  <text class="rd-hc" x="470" y="24">alice</text>
  <text class="rd-hc" x="556" y="24">bob</text>
  <text class="rd-l" x="8" y="54">round 1</text>
  <rect class="rd-a" x="78" y="40" width="78.0" height="20"/>
  <text class="rd-in" x="85.0" y="54">alice 100</text>
  <text class="rd-n" x="470" y="54">+100</text>
  <text class="rd-n" x="556" y="54">+0</text>
  <text class="rd-l" x="8" y="84">round 2</text>
  <rect class="rd-a" x="78" y="70" width="78.0" height="20"/>
  <text class="rd-in" x="85.0" y="84">alice 100</text>
  <rect class="rd-b" x="156.0" y="70" width="78.0" height="20"/>
  <text class="rd-in" x="163.0" y="84">bob 100</text>
  <text class="rd-n" x="470" y="84">+50</text>
  <text class="rd-n" x="556" y="84">+50</text>
  <text class="rd-l" x="8" y="114">round 3</text>
  <rect class="rd-a" x="78" y="100" width="78.0" height="20"/>
  <text class="rd-in" x="85.0" y="114">alice 100</text>
  <rect class="rd-b" x="156.0" y="100" width="78.0" height="20"/>
  <text class="rd-in" x="163.0" y="114">bob 100</text>
  <text class="rd-n" x="470" y="114">+50</text>
  <text class="rd-n" x="556" y="114">+50</text>
  <text class="rd-l" x="8" y="144">round 4</text>
  <rect class="rd-a" x="78" y="130" width="234.0" height="20"/>
  <text class="rd-in" x="85.0" y="144">alice 300</text>
  <rect class="rd-b" x="312.0" y="130" width="78.0" height="20"/>
  <text class="rd-in" x="319.0" y="144">bob 100</text>
  <text class="rd-n" x="470" y="144">+75</text>
  <text class="rd-n" x="556" y="144">+25</text>
  <path class="rd-rule" d="M8,162 L556,162"/>
  <text class="rd-l" x="8" y="182">after four rounds</text>
  <text class="rd-tot" x="470" y="182">275</text>
  <text class="rd-tot" x="556" y="182">125</text>
</svg>
<figcaption>The bar is the pool, and the two numbers are what that round paid.</figcaption>
</figure>

Those four rows are the whole requirement. The rest of this post is what they cost: every one of
them has to come out without the pool ever reading the list of people in it, and without anything
running between one round and the next.

## What the pool has to guarantee

Written as a rule those rows are four lines, and one stored number has to hold all four of them up:

- whoever holds x% of the pool receives x% of what the pool earns
- earnings are settled once per period, and nothing is paid out inside one
- the amount owed to a staker is correct at **any** block, not only just after a settlement
- what every staker is owed adds up to what the pool received, give or take what integer division
  leaves behind

A period is the system's own name for a round: a fixed stretch of its clock, and the unit the last
three lines are about.

Two more facts, and nobody chose them. They come from the chain:

- the block number grows for as long as the chain runs
- the number of stakers has no ceiling, and the product is not allowed to give it one

The staker count with no ceiling is what kills the answer everybody reaches for first.

## The loop dies at 1,274

The first answer anyone gives is a loop: walk the list, work out each share, credit it. On an
ordinary backend that is a job that runs for a few seconds and nobody thinks about it again.

On a chain that loop has to live inside one function call, and every step of it is charged. So I
priced it. `test/Loop.t.sol` is the cheapest version of that loop that could possibly work, no
transfers and no fees, one number written per staker, run at four staker counts.

<figure class="diagram">
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img"
     aria-label="A line chart. A straight teal line rises from the origin: the gas one call spends paying n stakers, about twenty three and a half thousand each. A dashed grey horizontal line marks the thirty million gas budget given to the call, and the teal line crosses it at one thousand two hundred and seventy four stakers, marked with a dot and a dashed drop to the axis. Past that point the line keeps climbing above the budget, where the call no longer fits.">
  <style>.lg-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.lg-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.lg-tl{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.lg-mu{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73}.lg-pt{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}.lg-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.lg-gd{stroke:#b8b4ab;stroke-width:1.2;fill:none;stroke-dasharray:3 3}.lg-line{stroke:#2f7d75;stroke-width:1.8;fill:none}.lg-cross{stroke:#2f7d75;stroke-width:1.2;fill:none;stroke-dasharray:3 3}.lg-dot{fill:#2f7d75}</style>
  <text class="lg-ml" x="74" y="24">gas to pay n stakers in one call &#8776; n &#215; 23.5k</text>
  <path class="lg-gd" d="M74,67.7 L556,67.7"/>
  <text class="lg-mu" x="80" y="61.7">30,000,000 gas, the budget this call is given</text>
  <path class="lg-cross" d="M483.4,206 L483.4,67.6"/>
  <path class="lg-line" d="M74.0,205.9 L556.0,43.1"/>
  <circle class="lg-dot" cx="483.4" cy="67.6" r="3.2"/>
  <text class="lg-pt" x="417.4" y="196">1,274 stakers</text>
  <path class="lg-ax" d="M74,32 L74,206 L556,206"/>
  <text class="lg-t" transform="rotate(-90 20 123)" x="20" y="123">gas used</text>
  <text class="lg-t" x="315" y="238">stakers paid in one call</text>
</svg>
<figcaption>Gas grows straight with the staker count, and the call has one budget.</figcaption>
</figure>

The budget is the wall, not the staker count. Past it the call does not pay
some of them and stop. It runs out of gas, the transaction reverts, and nobody was paid at all.

Splitting the loop over several transactions moves the problem. The pool now sits half paid between
batches, stakes can move while it is in that state, and someone has to send and fund every batch.

<details class="tryit" markdown="1">
<summary>The loop, and the run that produced that line</summary>

```solidity
/* test/Loop.t.sol */
contract PayEveryone {
  mapping(address staker => uint256 amount) public owed;
  address[] public stakers;

  function join(uint256 count) external {
    for (uint256 i; i < count; ++i) {
      stakers.push(address(uint160(i + 1)));
    }
  }

  function payAll(uint256 perStaker) external {
    for (uint256 i; i < stakers.length; ++i) {
      owed[stakers[i]] += perStaker;
    }
  }
}

contract LoopTest is Test {
  uint256 constant BUDGET = 30_000_000;

  function _gasToPay(uint256 count) internal returns (uint256) {
    PayEveryone pool = new PayEveryone();
    pool.join(count);
    uint256 before = gasleft();
    pool.payAll(1 ether);
    return before - gasleft();
  }

  function test_OneCallCannotPayEveryone() public {
    uint256 used;
    for (uint256 count = 10; count <= 10_000; count *= 10) {
      used = _gasToPay(count);
      console.log("stakers", count, "gas", used);
    }
    console.log("fits in a 30M budget:", BUDGET / (used / 10_000), "stakers");
  }
}
```

```
$ forge test -vv --match-path test/Loop.t.sol
# trimmed: the compile banner above this, and the pass count below it

Ran 1 test for test/Loop.t.sol:LoopTest
[PASS] test_OneCallCannotPayEveryone() (gas: 514316061)
Logs:
  stakers 10 gas 259283
  stakers 100 gas 2377883
  stakers 1000 gas 23563883
  stakers 10000 gas 235423883
  fits in a 30M budget: 1274 stakers
```

</details>

## Nothing runs on a schedule either

The second answer is a schedule. There is no schedule. A chain has no cron, no
timer and no background thread, so waiting is the whole experiment: let time pass, then read the
figure again.

```
$ forge test -vv --match-path test/Schedule.t.sol
# trimmed: the compile banner above this, and the pass count below it

Ran 1 test for test/Schedule.t.sol:ScheduleTest
[PASS] test_TimeAloneCreditsNobody() (gas: 315839)
Logs:
  after one payout             100
  a year and 2.5m blocks later 100
  after one more transaction   200
```

A year goes by between the payout and the second read, and two and a half million blocks are mined
in it. Time is not an event here, and neither is a block.

Only the transaction moves the number. So "pay everyone at midnight" needs a machine outside the
chain, awake and funded, and rewards stop when it does.

<details class="tryit" markdown="1">
<summary>The test that waited a year</summary>

```solidity
/* test/Schedule.t.sol */
contract ScheduleTest is Test {
  MockStaking pool;
  address POOL = makeAddr("pool");
  address ALICE = makeAddr("alice");

  function setUp() public {
    pool = new MockStaking(POOL);
    pool.firstEverWrapup();
    pool.stake(ALICE, 100 ether);
    pool.increasePeriod();
  }

  function test_TimeAloneCreditsNobody() public {
    pool.increaseReward(100 ether);
    pool.endPeriod();
    console.log("after one payout            ", pool.getRewardById(POOL, ALICE) / 1 ether);

    vm.warp(block.timestamp + 365 days);
    vm.roll(block.number + 2_500_000);
    console.log("a year and 2.5m blocks later", pool.getRewardById(POOL, ALICE) / 1 ether);

    pool.increaseReward(100 ether);
    pool.endPeriod();
    console.log("after one more transaction  ", pool.getRewardById(POOL, ALICE) / 1 ether);
  }
}
```

`vm.warp` sets the block timestamp and `vm.roll` sets the block number, both without producing a
transaction, which is exactly the thing being tested.

</details>

What the contract acts on instead is arrival: a reward handed to a pool, or a staker moving money.
Everything between those moments has to be a read.

The system does have one loop, and where it is allowed is the point. The contract that closes a
period walks the pools and hands each one its reward. The system caps that list itself: one entry
per operator in the active set, and the contract reads the size of that set from ([`maxValidatorNumber`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/validator/storage-fragments/ValidatorInfoStorageV2.sol#L118-L120))
rather than something a user can grow.

The stakers behind those pools are the opposite. Anyone can join, so their count is whatever the
market decides, and no loop ever touches them. One set is capped by the system's own rules, the
other grows with the market, and only the first kind can be walked. Which leaves the accounting
with four moments and nothing in between, and everything for the rest of this post happens at one of
them:

- **a stake changes**, because somebody added to a position or took part of it out
- **the pool is paid**, once at the end of a period, by the contract that closes the period
- **somebody claims**, and tokens actually move
- **somebody reads**, which is a question, not a transaction

The first three are transactions, and somebody pays for each one. The fourth is not a transaction at
all, and that gap is where the number on your screen comes from.

## A rate for every period

The arithmetic must never mention how many stakers there are. The move is the one any backend
reaches for when a per-row total is too expensive to recompute: keep one running total, and give
every row the value that total had the last time you looked at it.

Start with one period and one quantity. The **rate** for a period is what a single staked token
earned in it:

`rate = the pool's reward for that period / everything staked in that period`

- on top, what the pool was handed when the period closed, for everyone in it at once
- underneath, the pool's total stake, not yours

One division, and it does not care how many people are in the pool.

Hold a reward in your head as an area. Width is what you had staked, height is the rate for that
stretch of time, and the rectangle is what you are owed. Three of the four moments above are visible
in one picture: the pool is paid, you claim, the pool is paid again.

<figure class="steps">
<div class="steps-frames">
<svg viewBox="0 0 620 262" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 1. One teal rectangle under a bracket reading pending, as wide as the stake held in the first period and as tall as that period's rate."><style>.ra-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.ra-tl{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.ra-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.ra-m{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b;text-anchor:middle}.ra-mu{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73;text-anchor:end}.ra-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.ra-gd{stroke:#b8b4ab;stroke-width:1;fill:none;stroke-dasharray:3 3}.ra-old{fill:#f1efea;stroke:#b8b4ab;stroke-width:1.2}.ra-new{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.ra-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><text class="ra-ml" x="74" y="20">rate = the pool&#8217;s reward for that period &#247; everything staked in it</text><rect class="ra-new" x="74" y="166" width="70" height="40"/><path class="ra-gd" d="M74,166 L144,166"/><text class="ra-mu" x="66" y="170">rate 1</text><path class="ra-gd" d="M75,215 L75,221 M75,218 L143,218 M143,215 L143,221"/><text class="ra-t" x="109" y="234">balance 1</text><path class="ra-ax" d="M76,83 L76,78 L142,78 L142,83"/><text class="ra-t" x="109" y="73">pending</text><path class="ra-ax" d="M74,38 L74,206 L556,206"/><text class="ra-t" transform="rotate(-90 20 126)" x="20" y="126">reward per token</text><text class="ra-out" x="74" y="252">the pool is paid: one rectangle, and it is yours</text></svg>
<svg viewBox="0 0 620 262" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 2. A second rectangle joins the first under the same pending bracket, with its own width and its own height."><style>.ra-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.ra-tl{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.ra-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.ra-m{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b;text-anchor:middle}.ra-mu{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73;text-anchor:end}.ra-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.ra-gd{stroke:#b8b4ab;stroke-width:1;fill:none;stroke-dasharray:3 3}.ra-old{fill:#f1efea;stroke:#b8b4ab;stroke-width:1.2}.ra-new{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.ra-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><text class="ra-ml" x="74" y="20">rate = the pool&#8217;s reward for that period &#247; everything staked in it</text><rect class="ra-new" x="74" y="166" width="70" height="40"/><rect class="ra-new" x="144" y="122" width="110" height="84"/><path class="ra-gd" d="M74,166 L144,166"/><text class="ra-mu" x="66" y="170">rate 1</text><path class="ra-gd" d="M75,215 L75,221 M75,218 L143,218 M143,215 L143,221"/><text class="ra-t" x="109" y="234">balance 1</text><path class="ra-gd" d="M74,122 L254,122"/><text class="ra-mu" x="66" y="126">rate 2</text><path class="ra-gd" d="M145,215 L145,221 M145,218 L253,218 M253,215 L253,221"/><text class="ra-t" x="199" y="234">balance 2</text><path class="ra-ax" d="M76,83 L76,78 L252,78 L252,83"/><text class="ra-t" x="164" y="73">pending</text><path class="ra-ax" d="M74,38 L74,206 L556,206"/><text class="ra-t" transform="rotate(-90 20 126)" x="20" y="126">reward per token</text><text class="ra-out" x="74" y="252">paid again: your pending reward is the area of both</text></svg>
<svg viewBox="0 0 620 262" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 3. Both rectangles are now grey under a bracket reading claimed. Neither changed size or moved."><style>.ra-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.ra-tl{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.ra-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.ra-m{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b;text-anchor:middle}.ra-mu{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73;text-anchor:end}.ra-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.ra-gd{stroke:#b8b4ab;stroke-width:1;fill:none;stroke-dasharray:3 3}.ra-old{fill:#f1efea;stroke:#b8b4ab;stroke-width:1.2}.ra-new{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.ra-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><text class="ra-ml" x="74" y="20">rate = the pool&#8217;s reward for that period &#247; everything staked in it</text><rect class="ra-old" x="74" y="166" width="70" height="40"/><rect class="ra-old" x="144" y="122" width="110" height="84"/><path class="ra-gd" d="M74,166 L144,166"/><text class="ra-mu" x="66" y="170">rate 1</text><path class="ra-gd" d="M75,215 L75,221 M75,218 L143,218 M143,215 L143,221"/><text class="ra-t" x="109" y="234">balance 1</text><path class="ra-gd" d="M74,122 L254,122"/><text class="ra-mu" x="66" y="126">rate 2</text><path class="ra-gd" d="M145,215 L145,221 M145,218 L253,218 M253,215 L253,221"/><text class="ra-t" x="199" y="234">balance 2</text><path class="ra-ax" d="M76,83 L76,78 L252,78 L252,83"/><text class="ra-t" x="164" y="73">claimed</text><path class="ra-ax" d="M74,38 L74,206 L556,206"/><text class="ra-t" transform="rotate(-90 20 126)" x="20" y="126">reward per token</text><text class="ra-out" x="74" y="252">you claim: the tokens leave, the rectangles stay where they are</text></svg>
<svg viewBox="0 0 620 262" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Step 4. Two more teal rectangles stand to the right of the two grey ones, under a second bracket reading pending. The grey pair is untouched."><style>.ra-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.ra-tl{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.ra-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.ra-m{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b;text-anchor:middle}.ra-mu{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73;text-anchor:end}.ra-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.ra-gd{stroke:#b8b4ab;stroke-width:1;fill:none;stroke-dasharray:3 3}.ra-old{fill:#f1efea;stroke:#b8b4ab;stroke-width:1.2}.ra-new{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}.ra-out{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#2f7d75}</style><text class="ra-ml" x="74" y="20">rate = the pool&#8217;s reward for that period &#247; everything staked in it</text><rect class="ra-old" x="74" y="166" width="70" height="40"/><rect class="ra-old" x="144" y="122" width="110" height="84"/><rect class="ra-new" x="254" y="150" width="95" height="56"/><rect class="ra-new" x="349" y="136" width="85" height="70"/><path class="ra-gd" d="M74,166 L144,166"/><text class="ra-mu" x="66" y="170">rate 1</text><path class="ra-gd" d="M75,215 L75,221 M75,218 L143,218 M143,215 L143,221"/><text class="ra-t" x="109" y="234">balance 1</text><path class="ra-gd" d="M74,122 L254,122"/><text class="ra-mu" x="66" y="126">rate 2</text><path class="ra-gd" d="M145,215 L145,221 M145,218 L253,218 M253,215 L253,221"/><text class="ra-t" x="199" y="234">balance 2</text><path class="ra-gd" d="M74,150 L349,150"/><text class="ra-mu" x="66" y="154">rate 3</text><path class="ra-gd" d="M255,215 L255,221 M255,218 L348,218 M348,215 L348,221"/><text class="ra-t" x="301.5" y="234">balance 3</text><path class="ra-gd" d="M74,136 L434,136"/><text class="ra-mu" x="66" y="140">rate 4</text><path class="ra-gd" d="M350,215 L350,221 M350,218 L433,218 M433,215 L433,221"/><text class="ra-t" x="391.5" y="234">balance 4</text><path class="ra-ax" d="M76,83 L76,78 L252,78 L252,83"/><text class="ra-t" x="164" y="73">claimed</text><path class="ra-ax" d="M256,83 L256,78 L432,78 L432,83"/><text class="ra-t" x="344" y="73">pending</text><path class="ra-ax" d="M74,38 L74,206 L556,206"/><text class="ra-t" transform="rotate(-90 20 126)" x="20" y="126">reward per token</text><text class="ra-out" x="74" y="252">paid again: pending starts over, beside what you already took</text></svg>
</div>
<figcaption>A claim does not erase a rectangle. It moves the line between grey and teal.</figcaption>
</figure>

The third answer is to keep that rate for every block rather than every period, and add them up
when somebody asks. Written out, a read returns every rate the pool has paid, multiplied by what you
held at the time, minus whatever you have already taken:

`reward = sum over every block of (that block's rate x your balance then) - what you claimed`

The formula is honest and it is unusable, and the design document works it through only to throw it
away. Two lines from the constraints above sink it. The block number grows for as long as the
chain runs, so a rate per block is a row that is never finished. And your balance at some arbitrary
past block is not stored anywhere, so every term in that sum has to walk backwards to the last block
you changed anything.

Nothing of that shape was ever deployed, so there is no source to point at for it. Here it is
written out, and here is what one read costs as the blocks pile up:

```solidity
/* test/PerBlock.t.sol */
contract PerBlockRps {
  mapping(address staker => mapping(uint256 blockNo => uint256 amount)) public balance;
  mapping(uint256 blockNo => uint256 rate) public rate;
  uint256 public totalBalance;

  function onBalanceChange(address u, uint256 i, uint256 newBalance, uint256 oldBalance) external {
    balance[u][i] = newBalance;
    totalBalance = totalBalance + newBalance - oldBalance;
  }

  function onRewardRecorded(uint256 i, uint256 reward) external {
    rate[i] = reward / totalBalance;
  }

  function getUserReward(address u, uint256 upTo) external view returns (uint256 amount) {
    for (uint256 j; j < upTo; ++j) {
      uint256 k = j;
      while (k > 0 && balance[u][k] == 0) {
        --k;
      }
      amount += rate[j] * balance[u][k];
    }
  }
}
```

```
$ forge test -vv --match-path test/PerBlock.t.sol
# trimmed: the compile banner above this, and the pass count below it

Ran 1 test for test/PerBlock.t.sol:PerBlockTest
[PASS] test_ReadingCostsMoreEveryBlock() (gas: 150063745)
Logs:
  blocks 50 gas to read one reward 1423179
             and per block that is 28463
  blocks 100 gas to read one reward 5531654
             and per block that is 55316
  blocks 200 gas to read one reward 21811104
             and per block that is 109055
  blocks 400 gas to read one reward 86620004
             and per block that is 216550
```

Read the second number in each pair. The cost of one block doubles every time the block count
doubles, which is what a sum that walks a walk looks like from outside. Four hundred blocks in, one
read costs 86 million gas, against the 30 million the whole call was given earlier in this post. A
pool that has been open for four hundred blocks has barely opened.

<details class="tryit" markdown="1">
<summary>The harness that produced those four pairs</summary>

```solidity
/* test/PerBlock.t.sol */
contract PerBlockTest is Test {
  address ALICE = makeAddr("alice");

  function _gasToRead(uint256 blocks) internal returns (uint256) {
    PerBlockRps pool = new PerBlockRps();
    pool.onBalanceChange(ALICE, 0, 100 ether, 0);
    for (uint256 i; i < blocks; ++i) {
      pool.onRewardRecorded(i, 100 ether);
    }
    uint256 before = gasleft();
    pool.getUserReward(ALICE, blocks);
    return before - gasleft();
  }

  function test_ReadingCostsMoreEveryBlock() public {
    for (uint256 blocks = 50; blocks <= 400; blocks *= 2) {
      uint256 used = _gasToRead(blocks);
      console.log("blocks", blocks, "gas to read one reward", used);
      console.log("           and per block that is", used / blocks);
    }
  }
}
```

</details>

## One number for the pool, not one per staker

Stop storing the things being summed. Store the total instead, and call it the **meter**:

`meter = 0 when the pool opens`

`meter = meter + that period's rate, each time a period closes`

- it only ever goes up, so any two readings of it can be subtracted
- it is one number for the whole pool, whatever the rate was and however many people were in it

```solidity
/* _recordRewards in RewardCalculation.sol */
      // The rps is 0 if no one stakes for the pool
      rps = _pool.shares.inner == 0 ? 0 : (rewards[i] * 1e18) / _pool.shares.inner;
      aRps[i - count] = _pool.aRps += rps;
      _accumulatedRps[poolId][period] = PeriodWrapper(_pool.aRps, period);
      _pool.shares.inner = stakingTotal;
```

[`_recordRewards`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/RewardCalculation.sol#L210-L214)

The code calls the meter `aRps`, for accumulated reward per share, and farming pools call the same
variable `accRewardPerShare`. I am going to keep saying meter. Solidity has no floating point, so
the contract multiplies the reward by `1e18` before the division and divides it back out when a
claim is read, which is how you carry a fraction in an integer.

A meter reading on its own is not enough to store, because storage in Solidity has no empty state.
A slot nobody ever wrote reads back as zero, exactly like a slot someone wrote a zero into. A period
that paid nothing is not the same case as a period the pool never reached. So the
`_accumulatedRps` line writes the period number alongside the value, and nothing here is kept as a
bare number.

```solidity
/* PeriodWrapper in PeriodWrapperConsumer.sol */
  struct PeriodWrapper {
    // Inner value.
    uint256 inner;
    // Last period number that the info updated.
    uint256 lastPeriod;
  }
```

[`PeriodWrapper`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/interfaces/consumers/PeriodWrapperConsumer.sol#L5-L10)

A meter on its own pays nobody. Each staker also keeps a reading: what the meter said the last time
they touched the pool. What they have earned since is their balance times the difference between
that reading and the meter now. The sum is gone, and with it any need to know how many of them
there are.

<figure class="diagram">
<svg viewBox="0 0 620 262" xmlns="http://www.w3.org/2000/svg" role="img"
     aria-label="A staircase of four rectangles, one per visit, each wider and taller than the last. Each is the whole of the staker's balance times the meter's reading at that visit, marked underneath as balance one to balance four and on the left as meter one to meter four. The lower part of each rectangle is grey, up to the meter reading kept at the previous visit, and the band on top is teal. The teal band on the last rectangle is labelled new reward.">
  <style>.ra-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.ra-tl{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.ra-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.ra-m{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b;text-anchor:middle}.ra-mu{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73;text-anchor:end}.ra-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.ra-gd{stroke:#b8b4ab;stroke-width:1;fill:none;stroke-dasharray:3 3}.ra-old{fill:#f1efea;stroke:#b8b4ab;stroke-width:1.2}.ra-new{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}</style>
  <text class="ra-ml" x="74" y="20">new reward = your balance &#215; (the meter now &#8722; your last reading)</text>
  <rect class="ra-new" x="74" y="162" width="95" height="44"/>
  <rect class="ra-old" x="169" y="162" width="120" height="44"/>
  <rect class="ra-new" x="169" y="130" width="120" height="32"/>
  <rect class="ra-old" x="289" y="130" width="100" height="76"/>
  <rect class="ra-new" x="289" y="88" width="100" height="42"/>
  <rect class="ra-old" x="389" y="88" width="135" height="118"/>
  <rect class="ra-new" x="389" y="48" width="135" height="40"/>
  <text class="ra-m" x="456.5" y="72">new reward</text>
  <path class="ra-gd" d="M74,162 L169,162"/>
  <text class="ra-mu" x="66" y="166">meter 1</text>
  <path class="ra-gd" d="M75,215 L75,221 M75,218 L168,218 M168,215 L168,221"/>
  <text class="ra-t" x="121.5" y="234">balance 1</text>
  <path class="ra-gd" d="M74,130 L289,130"/>
  <text class="ra-mu" x="66" y="134">meter 2</text>
  <path class="ra-gd" d="M170,215 L170,221 M170,218 L288,218 M288,215 L288,221"/>
  <text class="ra-t" x="229" y="234">balance 2</text>
  <path class="ra-gd" d="M74,88 L389,88"/>
  <text class="ra-mu" x="66" y="92">meter 3</text>
  <path class="ra-gd" d="M290,215 L290,221 M290,218 L388,218 M388,215 L388,221"/>
  <text class="ra-t" x="339" y="234">balance 3</text>
  <path class="ra-gd" d="M74,48 L524,48"/>
  <text class="ra-mu" x="66" y="52">meter 4</text>
  <path class="ra-gd" d="M390,215 L390,221 M390,218 L523,218 M523,215 L523,221"/>
  <text class="ra-t" x="456.5" y="234">balance 4</text>
  <path class="ra-ax" d="M74,38 L74,206 L556,206"/>
  <text class="ra-t" x="315" y="252">every visit reads two numbers and subtracts them</text>
</svg>
<figcaption>Grey was counted at the visit before. Teal is what this visit is owed.</figcaption>
</figure>

```solidity
/* UserRewardFields in IRewardPool.sol */
  struct UserRewardFields {
    // Recorded reward amount.
    uint256 debited;
    // The last accumulated of the amount rewards per share (one unit staking) that the info updated.
    uint256 aRps;
    // Lowest staking amount in the period.
    uint256 lowestAmount;
    // Last period number that the info updated.
    uint256 lastPeriod;
  }
```

[`UserRewardFields`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/interfaces/staking/IRewardPool.sol#L10-L19)

Four numbers per staker, and `aRps` is the reading. It is stored on its own rather than already
multiplied by the balance, which is what a farming pool keeps under the name `rewardDebt`. That
difference is what the lowest-balance rule below needs, because there two different balances are
multiplied against the same reading, and a stored product could only ever have held one of them.

This is also why the figure on the screen moves without anyone sending a transaction. Reading it is
a `view` call, which runs on a node and changes no state, so it is right at whatever block that node
is at. The read touches four numbers belonging to that staker and two belonging to the pool, and
nobody else's. Nothing is settled first and nobody has to claim for the number to be correct.

What none of those six numbers can do is pay anybody.

## Changing your stake does not pay you

You have been in the pool for a period and a reward is sitting against your name. Now you want to
add more tokens. Here that transaction adds tokens. Nothing else moves: the number against your
name is the same after it as before, and it stays there until you go and take it.

```
$ forge test -vv --match-path test/Claim.t.sol
# trimmed: the compile banner above this, and the pass count below it

Ran 1 test for test/Claim.t.sol:ClaimTest
[PASS] test_MovingStakeLeavesTheRewardWhereItIs() (gas: 246086)
Logs:
  earned over one period     100
  after staking 100 more     100
  after taking 150 back out  100
  the claim call hands over  100
  and the pool now owes      0
```

Two of those lines move money in and out of the pool. One adds to the position, the other takes out
more than the addition put in. The reward does not flinch at either. The claim is the only
call that pays anything, and it is the only one that asked to. Taking money out does change what the
*next* period pays, which is the next section. What it cannot do is settle what the pool already
owes you.

The oldest and most copied reward pool works the other way round. In the original farming contract,
depositing harvests first:

```solidity
/* deposit in MasterChef.sol */
    // Deposit LP tokens to MasterChef for SUSHI allocation.
    function deposit(uint256 _pid, uint256 _amount) public {
        PoolInfo storage pool = poolInfo[_pid];
        UserInfo storage user = userInfo[_pid][msg.sender];
        updatePool(_pid);
        if (user.amount > 0) {
            uint256 pending =
                user.amount.mul(pool.accSushiPerShare).div(1e12).sub(
                    user.rewardDebt
                );
            safeSushiTransfer(msg.sender, pending);
        }
        pool.lpToken.safeTransferFrom(
            address(msg.sender),
            address(this),
            _amount
        );
        user.amount = user.amount.add(_amount);
        user.rewardDebt = user.amount.mul(pool.accSushiPerShare).div(1e12);
        emit Deposit(msg.sender, _pid, _amount);
    }
```

[`deposit`](https://github.com/sushiswap/masterchef/blob/4153a98c34e06ee3c373fcff566d1048dcd01666/contracts/MasterChef.sol#L233-L253)

So you are paid in a block you did not pick, at whatever the token is worth that day and in whatever
tax year that block falls in. That is the cost you can see. The one underneath it is in a pair of
lines. The payout goes out through `safeSushiTransfer`, which sends whichever is smaller, what you
are owed or what the contract is holding:

```solidity
/* safeSushiTransfer in MasterChef.sol */
    // Safe sushi transfer function, just in case if rounding error causes pool to not have enough SUSHIs.
    function safeSushiTransfer(address _to, uint256 _amount) internal {
        uint256 sushiBal = sushi.balanceOf(address(this));
        if (_amount > sushiBal) {
            sushi.transfer(_to, sushiBal);
        } else {
            sushi.transfer(_to, _amount);
        }
    }
```

[`safeSushiTransfer`](https://github.com/sushiswap/masterchef/blob/4153a98c34e06ee3c373fcff566d1048dcd01666/contracts/MasterChef.sol#L282-L290)

That is the first line. The second is back in `deposit`, which writes `user.rewardDebt` from the
full entitlement whatever the transfer actually managed to send. Anything the cap held back stops
being owed the moment that line runs. It is recorded as paid, and the next `pending` is measured
from there. The comment says the cap is for rounding dust, and the code has no way to tell rounding
dust from a pool that is short. Those two lines are read from the source, not run: the tests in this
post drive the Ronin contracts, and nothing here executes MasterChef.

Here the same moment rewrites the staker's record and moves nothing:

```solidity
/* _syncUserReward in RewardCalculation.sol */
  function _syncUserReward(address poolId, address user, uint256 newStakingAmount) internal {
  ...
    UserRewardFields storage _reward = _userReward[poolId][user];
    uint256 currentStakingAmount = _getStakingAmount(poolId, user);
    uint256 debited = _getReward(poolId, user, period, currentStakingAmount);

    if (_reward.debited != debited) {
      _reward.debited = debited;
      emit UserRewardUpdated(poolId, user, debited);
    }

    _syncMinStakingAmount(_pool, _reward, period, newStakingAmount, currentStakingAmount);
    _reward.aRps = _pool.aRps;
    _reward.lastPeriod = period;
  ...
```

[`_syncUserReward`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/RewardCalculation.sol#L94-L115)

`debited` is what you have earned, keeping upstream's name for it, and `aRps` is the reading it was
earned up to. There is no transfer in that function. There is none anywhere in the 238 lines of the
file: the code that works out what you are owed has no way to move a token.

Being paid is a separate call.
[`_claimReward`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/RewardCalculation.sol#L156-L167)
settles the ledger and hands the amount back to whoever called it. It zeroes `debited`, moves your
reading up to the meter, and sends nothing. Its own comment says that is the point of it. The
[`claimRewards`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/DelegatorStaking.sol#L70-L78)
a delegator calls does the settlement and then the transfer, and the transfer is the strict kind.
[`_sendRON`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/extensions/RONTransferHelper.sol#L27-L30)
reverts if the contract is short. `_transferRON` reverts if the recipient refuses the money. A
payout that cannot be made in full is not made at all, and the settlement is unwound with the rest
of the transaction. One design writes down what it wishes it had sent. This one cannot write
anything down unless the tokens went with it.

That farming contract's own second version
[dropped the forced harvest](https://github.com/sushiswap/masterchef/blob/4153a98c34e06ee3c373fcff566d1048dcd01666/contracts/MasterChefV2.sol#L210-L227)
too, and gave harvesting its own call, but the ledger and the payout still sit in one contract. What
is split here is not the calls, it is the layers: the arithmetic lives in a file with no way to send
anything, and the contracts above it decide when tokens move.

<details class="tryit" markdown="1">
<summary>The four lines above, and the contract they came from</summary>

```solidity
/* test/Claim.t.sol */
contract ClaimTest is Test {
  MockStaking pool;
  address POOL = makeAddr("pool");
  address ALICE = makeAddr("alice");

  function setUp() public {
    pool = new MockStaking(POOL);
    pool.firstEverWrapup();
    pool.stake(ALICE, 100 ether);
    pool.increasePeriod();
    pool.increaseReward(100 ether);
    pool.endPeriod();
  }

  function test_MovingStakeLeavesTheRewardWhereItIs() public {
    console.log("earned over one period    ", pool.getRewardById(POOL, ALICE) / 1 ether);

    pool.stake(ALICE, 100 ether);
    console.log("after staking 100 more    ", pool.getRewardById(POOL, ALICE) / 1 ether);

    pool.unstake(ALICE, 150 ether);
    console.log("after taking 150 back out ", pool.getRewardById(POOL, ALICE) / 1 ether);

    console.log("the claim call hands over ", pool.claimReward(ALICE) / 1 ether);
    console.log("and the pool now owes     ", pool.getRewardById(POOL, ALICE) / 1 ether);
  }
}
```

</details>

## Paid on the lowest balance you held

Rewards land at the end of a period, and the balance at that instant is public. So take your stake
out at the start of the period, use it somewhere else, and put it back one block before the wrap-up.
The meter notices nothing. It only ever sees the balance at the moment it is read.

So the user's record carries `lowestAmount`, and the contract pays on that instead of on the closing
balance. Putting the money back gains nothing. The low point is already written.

Lowering one staker's figure is only half of the change. The pool divides by a total, and if that
total still counts money that has left, the shares no longer total the reward and part of it has no
owner. So the same call moves both numbers.

```solidity
/* _syncMinStakingAmount in RewardCalculation.sol */
  /**
   * @dev Syncs the minimum staking amount of an user in the current period.
   */
  function _syncMinStakingAmount(
    PoolFields storage _pool,
    UserRewardFields storage _reward,
    uint256 latestPeriod,
    uint256 newStakingAmount,
    uint256 currentStakingAmount
  ) internal {
    if (_reward.lastPeriod < latestPeriod) {
      _reward.lowestAmount = currentStakingAmount;
    }

    uint256 lowestAmount = Math.min(_reward.lowestAmount, newStakingAmount);
    uint256 diffAmount = _reward.lowestAmount - lowestAmount;
    if (diffAmount > 0) {
      _reward.lowestAmount = lowestAmount;
      if (_pool.shares.inner < diffAmount) revert ErrInvalidPoolShare();
      _pool.shares.inner -= diffAmount;
    }
  }
```

[`_syncMinStakingAmount`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/RewardCalculation.sol#L122-L143)

Take the `_pool.shares.inner -= diffAmount` line away and the arithmetic still runs, but part of
the period's reward has no owner.

It is the same rule that set the timing in Alice and Bob's four rounds, and it does not treat the
two directions alike. Money added inside a period earns nothing for that period, because the
contract recorded the low point before it arrived. Money taken out lands on the period straight
away, because taking it out is what moves the low point.

A pool's shares are not the sum of what people hold. They are the sum of the lowest amounts people
held this period. Read `_getReward` against that: it is the function behind both the figure on
the screen and the transfer on a claim.

```solidity
/* _getReward in RewardCalculation.sol */
  /**
   * @dev Returns the reward amount that user claimable.
   */
  function _getReward(
    address poolId,
    address user,
    uint256 latestPeriod,
    uint256 latestStakingAmount
  ) internal view returns (uint256) {
    UserRewardFields storage _reward = _userReward[poolId][user];

    if (_reward.lastPeriod == latestPeriod) {
      return _reward.debited;
    }

    uint256 aRps;
    uint256 lastPeriodReward;
    PoolFields storage _pool = _stakingPool[poolId];
    PeriodWrapper storage _wrappedArps = _accumulatedRps[poolId][_reward.lastPeriod];

    if (_wrappedArps.lastPeriod > 0) {
      // Calculates the last period reward if the aRps at the period is set
      aRps = _wrappedArps.inner;
      lastPeriodReward = _reward.lowestAmount * (aRps - _reward.aRps);
    } else {
      // Fallbacks to the previous aRps in case the aRps is not set
      aRps = _reward.aRps;
    }

    uint256 newPeriodsReward = latestStakingAmount * (_pool.aRps - aRps);
    return _reward.debited + (lastPeriodReward + newPeriodsReward) / 1e18;
  }
```

[`_getReward`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/RewardCalculation.sol#L52-L83)

It adds two products, and they are the two cases rather than an optimisation. The period in which
the staker last moved money is paid on `lowestAmount`. Every whole period after it is paid on the
balance they hold now, because in those periods they moved nothing. One checkpoint, two balances,
which is why the checkpoint is stored on its own.

A fresh pool, Alice and Bob with one hundred tokens each. They sit through one period, and in the
next one Alice drops to ten and restores the ninety before the wrap-up. The contract under test is
upstream's own mock, so it inherits the deployed accounting unchanged:

```solidity
/* test/Rps.t.sol */
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { Test, console } from "forge-std/Test.sol";
import { MockStaking } from "dpos/mocks/MockStaking.sol";

contract RpsTest is Test {
  MockStaking pool;
  address POOL = makeAddr("pool");
  address ALICE = makeAddr("alice");
  address BOB = makeAddr("bob");

  function setUp() public {
    pool = new MockStaking(POOL);
    pool.firstEverWrapup();
  }

  function test_PaidOnTheLowestBalanceHeldInThePeriod() public {
    pool.stake(ALICE, 100 ether);
    pool.stake(BOB, 100 ether);
    pool.increasePeriod();

    // A period both of them sit through, so the split is the plain one.
    pool.increaseReward(100 ether);
    pool.endPeriod();
    console.log("period 1  alice", pool.getRewardById(POOL, ALICE));
    console.log("period 1  bob  ", pool.getRewardById(POOL, BOB));

    // Alice drops to 10 tokens inside the next period and restores it before the wrap-up.
    pool.unstake(ALICE, 90 ether);
    pool.stake(ALICE, 90 ether);
    pool.increaseReward(100 ether);
    pool.endPeriod();

    uint256 alice = pool.getRewardById(POOL, ALICE);
    uint256 bob = pool.getRewardById(POOL, BOB);
    console.log("period 2  alice", alice);
    console.log("period 2  bob  ", bob);
    console.log("unpaid        ", uint256(200 ether) - alice - bob);
  }
}
```

```
$ forge test -vv --match-path test/Rps.t.sol
# trimmed: the compile banner above this, and the pass count below it

Ran 1 test for test/Rps.t.sol:RpsTest
[PASS] test_PaidOnTheLowestBalanceHeldInThePeriod() (gas: 759725)
Logs:
  period 1  alice 50000000000000000000
  period 1  bob   50000000000000000000
  period 2  alice 59090909090909090900
  period 2  bob   140909090909090909000
  unpaid         100
```

The last line is a hundred wei, the remainder of a division that is not exact.

<figure class="diagram">
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img"
     aria-label="Two rectangles for one period, drawn to scale. Alice is ten wide because ten is the lowest she held, bob is one hundred wide, and the ninety alice removed is drawn as an empty dashed box outside the axis. The pool divides the reward by one hundred and ten rather than two hundred, so the height is larger and bob's area grows by what alice released.">
  <style>.ra-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.ra-tl{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.ra-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.ra-m{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b;text-anchor:middle}.ra-mu{font:11px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73;text-anchor:end}.ra-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.ra-gd{stroke:#b8b4ab;stroke-width:1;fill:none;stroke-dasharray:3 3}.ra-old{fill:#f1efea;stroke:#b8b4ab;stroke-width:1.2}.ra-new{fill:#e8f2f0;stroke:#2f7d75;stroke-width:1.4}</style>
  <text class="ra-ml" x="74" y="24">rate = 100 tokens &#247; 110, not &#247; 200</text>
  <rect class="ra-new" x="74" y="80" width="22" height="126"/>
  <rect class="ra-new" x="96" y="80" width="220" height="126"/>
  <rect class="ra-gd" x="316" y="80" width="198" height="126" fill="none"/>
  <text class="ra-m" x="206" y="143">90.90</text>
  <text class="ra-t" x="206" y="226">bob holds 100</text>
  <text class="ra-t" x="415" y="226">the 90 alice pulled out</text>
  <text class="ra-ml" x="106" y="68">alice, lowest 10, earns 9.09</text>
  <path class="ra-ax" d="M85,64 L85,77"/>
  <path class="ra-ax" d="M74,38 L74,206 L556,206"/>
  <text class="ra-t" transform="rotate(-90 20 126)" x="20" y="126">reward per token</text>
  <text class="ra-t" x="315" y="246">a shorter axis makes every rectangle left on it taller</text>
</svg>
<figcaption>Alice&#8217;s width drops to her lowest, and the axis drops with it.</figcaption>
</figure>

Bob moved nothing across both periods and is better off for it. The reward Alice released was not
burned and it did not stay in the contract. It went to the staker who did hold, because the divisor
moved at the same instant her own figure did.

## The pool that is not paid at all

In this system a pool belongs to a validator. A validator that is jailed or marked unavailable
forfeits the period, and so does everyone staked on it. The reason has nothing to do with
arithmetic.

A staker who did nothing wrong loses the period because the operator they picked misbehaved, and
the number sitting in their record goes with it.

This is where the design document and the deployed system part company. In the document I treated
slashing as an accounting problem and solved it inside the pool, in four moves:

- the reward splits into a pending pool and a settled pool
- new fields remember how much of each user's reward has settled
- the getter splits into three, for the claimable part, the pending part and the total
- the claimable one branches three ways, on where the user's last action fell

That last line is the one that costs. Time is now cut into stretches, a staker's last visit fell
into exactly one of them, and which one decided the formula.

<figure class="diagram">
<svg viewBox="0 0 620 250" xmlns="http://www.w3.org/2000/svg" role="img"
     aria-label="Three timelines, one under the other, sharing three regions marked slashed, settled and pending. Each timeline carries a cross showing where a staker's last visit fell: in the slashed region for case one, the settled region for case two, the pending region for case three. Each case is labelled with its own formula. The whole figure is drawn in grey because none of this code exists in the shipped contract.">
  <style>.sc-t{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73;text-anchor:middle}.sc-l{font:11.5px ui-sans-serif,system-ui,sans-serif;fill:#6b6b73}.sc-r{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#6b6b73}.sc-ml{font:11.5px ui-monospace,SFMono-Regular,Menlo,monospace;fill:#26262b}.sc-ax{stroke:#b8b4ab;stroke-width:1.3;fill:none}.sc-sep{stroke:#b8b4ab;stroke-width:1;fill:none;stroke-dasharray:3 3}.sc-x{stroke:#26262b;stroke-width:1.6;fill:none}</style>
  <text class="sc-ml" x="8" y="22">your last visit landed here</text>
  <text class="sc-t" x="155" y="46">slashed</text>
  <text class="sc-t" x="290" y="46">settled</text>
  <text class="sc-t" x="418" y="46">pending</text>
  <path class="sc-sep" d="M214,54 L214,214"/>
  <path class="sc-sep" d="M366,54 L366,214"/>
  <text class="sc-l" x="8" y="90">case 1</text>
  <path class="sc-ax" d="M96,86 L470,86"/>
  <path class="sc-x" d="M151,82 L159,90 M151,90 L159,82"/>
  <text class="sc-r" x="484" y="90">formula 1</text>
  <text class="sc-l" x="8" y="140">case 2</text>
  <path class="sc-ax" d="M96,136 L470,136"/>
  <path class="sc-x" d="M286,132 L294,140 M286,140 L294,132"/>
  <text class="sc-r" x="484" y="140">formula 2</text>
  <text class="sc-l" x="8" y="190">case 3</text>
  <path class="sc-ax" d="M96,186 L470,186"/>
  <path class="sc-x" d="M414,182 L422,190 M414,190 L422,182"/>
  <text class="sc-r" x="484" y="190">formula 3</text>
  <text class="sc-t" x="283" y="236">one question, three answers, and none of it was built</text>
</svg>
<figcaption>Where your last visit fell decided which formula answered you.</figcaption>
</figure>

None of that is in the contract. `RewardCalculation.sol` has no notion of slashing at all. Search it
for the word and there is nothing to find. The decision lives one contract higher up, in the code
that closes a period:

```solidity
/* the period wrap-up in CoinbaseExecution.sol */
    for (uint i; i < length; ++i) {
      vId = cids[i];
      treasury = _candidateInfo[vId].__shadowedTreasury;

      if (!_isJailedById(vId) && !_miningRewardDeprecatedById(vId, lastPeriod)) {
    ...
        delegatorBlockMiningRewards[i] = _delegatorMiningReward[vId];
    ...
      } else {
        _totalDeprecatedReward += _validatorMiningReward[vId] + _delegatorMiningReward[vId] + _fastFinalityReward[vId];
      }
    ...
```

[`_distributeRewardToTreasuriesAndCalculateTotalDelegatorsReward`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/validator/CoinbaseExecution.sol#L216-L235)

A jailed validator takes the `else` branch, so its entry in the rewards array is left at zero and
the money is added to a total that is recycled later. The staking contract is then handed a zero for
that pool, divides it by the total stake, and adds zero to the accumulator.

<p class="gloss">The decision sits one contract up.</p>

So none of what I wrote there shipped. It was deleted: two pools, three cases and a set of extra
fields, gone together, because the question *is this pool paid this period* was
answered before the pool was ever asked. A layer that cannot see slashing cannot get slashing wrong.

## What the pool never does

- It never reads the list of stakers, at any point, for any reason.
- It never runs on a timer, because nothing here runs on its own.
- It never stores a reward per staker. It stores one meter and four numbers each, and derives the
  reward when someone asks.
- It is never out of date. The figure is right at the block you read it at, with nothing settled
  first and nothing claimed.
- It never pays a staker for money that was not in the pool, because it prices the period on the
  lowest balance each of them held.
- It never learns what slashing is. A pool that must not be paid arrives with a reward of zero.
- It never pays you for changing your mind. Moving your stake and taking your reward are two
  separate calls.

## Sources

- [`RewardCalculation.sol`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/RewardCalculation.sol),
  the whole accounting layer, 238 lines
- [`IRewardPool.sol`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/interfaces/staking/IRewardPool.sol#L10-L26),
  the two records every claim is computed from
- [`CoinbaseExecution.sol`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/validator/CoinbaseExecution.sol#L216-L235),
  where a jailed validator loses the period
- [`DelegatorStaking.sol`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/ronin/staking/DelegatorStaking.sol#L70-L78),
  the claim that settles the ledger and then pays
- [`RONTransferHelper.sol`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/extensions/RONTransferHelper.sol#L27-L30),
  the transfer that reverts rather than sending less than it owes
- [`MockStaking.sol`](https://github.com/ronin-chain/dpos-contract/blob/348e2e364fb9d63faa116dd61ea71a1161578fd7/contracts/mocks/MockStaking.sol),
  upstream's mock, which is what the test here drives
- [`MasterChef.sol`](https://github.com/sushiswap/masterchef/blob/4153a98c34e06ee3c373fcff566d1048dcd01666/contracts/MasterChef.sol#L233-L253),
  the deposit that pays you whether you asked or not
- [`MasterChefV2.sol`](https://github.com/sushiswap/masterchef/blob/4153a98c34e06ee3c373fcff566d1048dcd01666/contracts/MasterChefV2.sol#L210-L227),
  the same deposit after the forced harvest came out
