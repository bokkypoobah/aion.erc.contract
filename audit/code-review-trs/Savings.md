# Savings

Source file [../../trs/contracts/Savings.sol](../../trs/contracts/Savings.sol).

<br />

<hr />

```javascript
// Copyright New Alchemy Limited, 2017. All rights reserved.
// BK Ok
pragma solidity >=0.4.10;

// BK Ok
contract Token {
    // BK Ok
	function transferFrom(address from, address to, uint amount) returns(bool);
	// BK Ok
	function transfer(address to, uint amount) returns(bool);
	// BK Ok
	function balanceOf(address addr) constant returns(uint);
}

// BK Ok
import {Owned} from '../../standard/contracts/Owned.sol';

/**
 * Savings is a contract that releases Tokens on a predefined
 * schedule, and allocates bonus tokens upon withdrawal on a
 * proportional basis, determined by the ratio of deposited tokens
 * to total owned tokens.
 *
 * The distribution schedule consists of a monthly withdrawal schedule
 * responsible for distribution 75% of the total savings, and a
 * one-off withdrawal event available before or at the start of the
 * withdrawal schedule, distributing 25% of the total savings.
 *
 * To be exact, upon contract deployment there may be a period of time in which
 * only the one-off withdrawal event is available, define this period of time as:
 * [timestamp(start), timestamp(startBlockTimestamp)),
 *
 * Then the periodic withdrawal range is defined as:
 * [timestamp(startBlockTimestamp), +inf)
 *
 * DO NOT SEND TOKENS TO THIS CONTRACT. Use the deposit() or depositTo() method.
 * As an exception, tokens transferred to this contract before locking are the
 * bonus tokens that are distributed.
 */
// BK Ok
contract Savings is Owned {
	/**
	 * Periods is the total monthly withdrawable amount, not counting the
	 * special withdrawal.
	 */
	// BK Ok
	uint public periods;

	/**
	 * t0special is an additional multiplier that determines what
	 * fraction of the total distribution is distributed in the
	 * one-off withdrawal event. It is used in conjunction with
	 * a periodic multiplier (p) to determine the total savings withdrawable
	 * to the user at that point in time.
	 *
	 * The value is not set, it is calculated based on periods
	 */
	// BK Ok
	uint public t0special;

    // BK Ok
	uint constant public intervalSecs = 30 days;
	// BK Ok
	uint constant public precision = 10 ** 18;

	/**
	 * Events
	 */
    // BK Next 2 Ok - Events
	event Withdraws(address indexed who, uint amount);
	event Deposit(address indexed who, uint amount);

    // BK Ok
	bool public inited;
    // BK Ok
	bool public locked;
	// BK Ok
	uint public startBlockTimestamp = 0;

    // BK Ok
	Token public token;

	// face value deposited by an address before locking
	// BK Ok
	mapping (address => uint) public deposited;

	// total face value deposited; sum of deposited
	// BK Ok
	uint public totalfv;

	// the total remaining value
	// BK Ok
	uint public remainder;

	/**
	 * Total tokens owned by the contract after locking, and possibly
	 * updated by the foundation after subsequent sales.
	 */
	// BK Ok
	uint public total;

	// the total value withdrawn
	// BK Ok
	mapping (address => uint256) public withdrawn;

	// BK Ok
	bool public nullified;

    // BK Ok
	modifier notNullified() { require(!nullified); _; }

    // BK Ok - Before lock and not started
	modifier preLock() { require(!locked && startBlockTimestamp == 0); _; }

	/**
	 * Lock called, deposits no longer available.
	 */
	// BK Ok - Note that this is not used
	modifier postLock() { require(locked); _; }

	/**
	 * Prestart, state is after lock, before start
	 */
	// BK Ok
	modifier preStart() { require(locked && startBlockTimestamp == 0); _; }

	/**
	 * Start called, the savings contract is now finalized, and withdrawals
	 * are now permitted.
	 */
	// BK Ok
	modifier postStart() { require(locked && startBlockTimestamp != 0); _; }

	/**
	 * Uninitialized state, before init is called. Mainly used as a guard to
	 * finalize periods and t0special.
	 */
	// BK Ok
	modifier notInitialized() { require(!inited); _; }

	/**
	 * Post initialization state, mainly used to guarantee that
	 * periods and t0special have been set properly before starting
	 * the withdrawal process.
	 */
	// BK Ok
	modifier initialized() { require(inited); _; }

	/**
	 * Revert under all conditions for fallback, cheaper mistakes
	 * in the future?
	 */
	// BK Ok
	function() {
	    // BK Ok
		revert();
	}

	/**
	 * Nullify functionality is intended to disable the contract.
	 */
	// BK NOTE - This function will disable all withdrawals and cannot be undone
	// BK Ok - Only owner can execute
	function nullify() onlyOwner {
	    // BK Ok
		nullified = true;
	}

	/**
	 * Initialization function, should be called after contract deployment. The
	 * addition of this function allows contract compilation to be simplified
	 * to one contract, instead of two.
	 *
	 * periods and t0special are finalized, and effectively invariant, after
	 * init is called for the first time.
	 */
	// BK Ok - Only owner can execute once at the start
	function init(uint _periods, uint _t0special) onlyOwner notInitialized {
	    // BK Ok
		require(_periods != 0);
		// BK Ok
		periods = _periods;
		// BK Ok
		t0special = _t0special;
	}

    // BK Ok - Only owner can execute once at the start
	function finalizeInit() onlyOwner notInitialized {
	    // BK Ok
		inited = true;
	}

    // BK Ok - Only owner can execute
	function setToken(address tok) onlyOwner {
	    // BK Ok
		token = Token(tok);
	}

	/**
	 * Lock is called by the owner to lock the savings contract
	 * so that no more deposits may be made.
	 */
	// BK Ok - Only owner can execute
	function lock() onlyOwner {
	    // BK Ok
		locked = true;
	}

	/**
	 * Starts the distribution of savings, it should be called
	 * after lock(), once all of the bonus tokens are send to this contract,
	 * and multiMint has been called.
	 */
	// BK Ok - Only owner can execute once at the start
	function start(uint _startBlockTimestamp) onlyOwner initialized preStart {
	    // BK Ok
		startBlockTimestamp = _startBlockTimestamp;
		// BK Ok
		uint256 tokenBalance = token.balanceOf(this);
		total = tokenBalance;
		remainder = tokenBalance;
	}

	/**
	 * Check withdrawal is live, useful for checking whether
	 * the savings contract is "live", withdrawal enabled, started.
	 */
	// BK Ok - Constant function
	function isStarted() constant returns(bool) {
	    // BK Ok
		return locked && startBlockTimestamp != 0;
	}

	// if someone accidentally transfers tokens to this contract,
	// the owner can return them as long as distribution hasn't started

	/**
	 * Used to refund users who accidentaly transferred tokens to this
	 * contract, only available before contract is locked
	 */
	// BK Ok - Only owner can execute, before contract locked
	function refundTokens(address addr, uint amount) onlyOwner preLock {
	    // BK Ok
		token.transfer(addr, amount);
	}


	/**
	 * Update the total balance, to be called in case of subsequent sales. Updates
	 * the total recorded balance of the contract by the difference in expected
	 * remainder and the current balance. This means any positive difference will
	 * be "recorded" into the contract, and distributed within the remaining
	 * months of the TRS.
	 */
	// BK Ok - Only owner can execute, after contract locked
	function updateTotal() onlyOwner postLock {
	    // BK Ok
		uint current = token.balanceOf(this);
		// BK Ok
		require(current >= remainder); // for sanity

        // BK Ok
		uint difference = (current - remainder);
		// BK Ok
		total += difference;
		// BK Ok
		remainder = current;
	}

	/**
	 * Calculates the monthly period, starting after the startBlockTimestamp,
	 * periodAt will return 0 for all timestamps before startBlockTimestamp.
	 *
	 * Therefore period 0 is the range of time in which we have called start(),
	 * but have not yet passed startBlockTimestamp. Period 1 is the
	 * first monthly period, and so-forth all the way until the last
	 * period == periods.
	 *
	 * NOTE: not guarded since no state modifications are made. However,
	 * it will return invalid data before the postStart state. It is
	 * up to the user to manually check that the contract is in
	 * postStart state.
	 */
	// BK Ok - Constant function
	function periodAt(uint _blockTimestamp) constant returns(uint) {
		/**
		 * Lower bound, consider period 0 to be the time between
		 * start() and startBlockTimestamp
		 */
		// BK Ok
		if (startBlockTimestamp > _blockTimestamp)
		    // BK Ok
			return 0;

		/**
		 * Calculate the appropriate period, and set an upper bound of
		 * periods - 1.
		 */
		// BK Ok
		uint p = ((_blockTimestamp - startBlockTimestamp) / intervalSecs) + 1;
		// BK Ok
		if (p > periods)
		    // BK Ok
			p = periods;
		// BK Ok
		return p;
	}

	// what withdrawal period are we in?
	// returns the period number from [0, periods)
	// BK OK - Constant function
	function period() constant returns(uint) {
	    // BK Ok
		return periodAt(block.timestamp);
	}

	// deposit your tokens to be saved
	//
	// the despositor must have approve()'d the tokens
	// to be transferred by this contract
	// BK Ok - Only owner can execute
	function deposit(uint tokens) onlyOwner notNullified {
	    // BK Ok
		depositTo(msg.sender, tokens);
	}


    // BK Ok - Only owner can execute
	function depositTo(address beneficiary, uint tokens) onlyOwner preLock notNullified {
	    // BK Ok
		require(token.transferFrom(msg.sender, this, tokens));
		// BK Ok
	    deposited[beneficiary] += tokens;
	    // BK Ok
		totalfv += tokens;
		// BK Ok - Log event
		Deposit(beneficiary, tokens);
	}

	// convenience function for owner: deposit on behalf of many
	// BK Ok
	function bulkDepositTo(uint256[] bits) onlyOwner {
	    // BK Ok
		uint256 lomask = (1 << 96) - 1;
		// BK Ok
		for (uint i=0; i<bits.length; i++) {
		    // BK Ok
			address a = address(bits[i]>>96);
            // BK Ok
			uint val = bits[i]&lomask;
            // BK Ok
			depositTo(a, val);
		}
	}

	// withdraw withdraws tokens to the sender
	// withdraw can be called at most once per redemption period
	// BK Ok - Anyone can call this
	function withdraw() notNullified returns(bool) {
	    // BK Ok
		return withdrawTo(msg.sender);
	}

	/**
	 * Calculates the fraction of total (one-off + monthly) withdrawable
	 * given the current timestamp. No guards due to function being constant.
	 * Will output invalid data until the postStart state. It is up to the user
	 * to manually confirm contract is in postStart state.
	 */
	// BK Ok - Constant function
	function availableForWithdrawalAt(uint256 blockTimestamp) constant returns (uint256) {
		/**
		 * Calculate the total withdrawable, giving a numerator with range:
		 * [0.25 * 10 ** 18, 1 * 10 ** 18]
		 */
		// BK Ok
		return ((t0special + periodAt(blockTimestamp)) * precision) / (t0special + periods);
	}

	/**
	 * Business logic of _withdrawTo, the code is separated this way mainly for
	 * testing. We can inject and test parameters freely without worrying about the
	 * blockchain model.
	 *
	 * NOTE: Since function is constant, no guards are applied. This function will give
	 * invalid outputs unless in postStart state. It is up to user to manually check
	 * that the correct state is given (isStart() == true)
	 */
	// BK Ok - Constant function
	function _withdrawTo(uint _deposit, uint _withdrawn, uint _blockTimestamp, uint _total) constant returns (uint) {
	    // BK Ok
		uint256 fraction = availableForWithdrawalAt(_blockTimestamp);

		/**
		 * There are concerns that the multiplication could possibly
		 * overflow, however this should not be the case if we calculate
		 * the upper bound based on our known parameters:
		 *
		 * Lets assume the minted token amount to be 500 million (reasonable),
		 * given a precision of 8 decimal places, we get:
		 * deposited[addr] = 5 * (10 ** 8) * (10 ** 8) = 5 * (10 ** 16)
		 *
		 * The max for fraction = 10 ** 18, and the max for total is
		 * also 5 * (10 ** 16).
		 *
		 * Therefore:
		 * deposited[addr] * fraction * total = 2.5 * (10 ** 51)
		 *
		 * The maximum for a uint256 is = 1.15 * (10 ** 77)
		 */
		// BK Ok
		uint256 withdrawable = ((_deposit * fraction * _total) / totalfv) / precision;

		// check that we can withdraw something
        // BK Ok
		if (withdrawable > _withdrawn) {
            // BK Ok
			return withdrawable - _withdrawn;
		}
        // BK Ok
		return 0;
	}

	/**
	 * Public facing withdrawTo, injects business logic with
	 * the correct model.
	 */
    // BK Ok
	function withdrawTo(address addr) postStart notNullified returns (bool) {
        // BK Ok
		uint _d = deposited[addr];
        // BK Ok
		uint _w = withdrawn[addr];

        // BK Ok
		uint diff = _withdrawTo(_d, _w, block.timestamp, total);

		// no withdrawal could be made
        // BK Ok
		if (diff == 0) {
            // BK Ok
			return false;
		}

		// check that we cannot withdraw more than max
        // BK Ok
		require((diff + _w) <= ((_d * total) / totalfv));

		// transfer and increment
        // BK Ok
		require(token.transfer(addr, diff));

        // BK Ok
		withdrawn[addr] += diff;
		remainder -= diff;
        // BK Ok - Log event
		Withdraws(addr, diff);
        // BK Ok
		return true;
	}

	// force withdrawal to many addresses
    // BK Ok
	function bulkWithdraw(address[] addrs) notNullified {
        // BK Ok
		for (uint i=0; i<addrs.length; i++)
            // BK Ok
			withdrawTo(addrs[i]);
	}

	// Code off the chain informs this contract about
	// tokens that were minted to it on behalf of a depositor.
	//
	// Note: the function signature here is known to New Alchemy's
	// tooling, which is why it is arguably misnamed.
    // BK Ok
	uint public mintingNonce;
    // BK Ok
	function multiMint(uint nonce, uint256[] bits) onlyOwner preLock {

        // BK Ok
		if (nonce != mintingNonce) return;
        // BK Ok
		mintingNonce += 1;
        // BK Ok
		uint256 lomask = (1 << 96) - 1;
        // BK Ok
		uint sum = 0;
        // BK Ok
		for (uint i=0; i<bits.length; i++) {
            // BK Ok
			address a = address(bits[i]>>96);
            // BK Ok
			uint value = bits[i]&lomask;
            // BK Ok
			deposited[a] += value;
            // BK Ok
			sum += value;
            // BK Ok
			Deposit(a, value);
		}
        // BK Ok
		totalfv += sum;
	}
}

```
