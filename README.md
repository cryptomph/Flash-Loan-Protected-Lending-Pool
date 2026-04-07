# Flash-Loan-Protected-Lending-Pool
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/token/ERC20/IERC20.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/access/Ownable.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/security/ReentrancyGuard.sol";

contract FlashProtectedLending is Ownable, ReentrancyGuard {
    IERC20 public lendingToken;
    uint256 public totalSupplied;
    mapping(address => uint256) public balances;
    mapping(address => uint256) public borrows;

    uint256 public flashFee = 9; // 0.09% fee

    event Supplied(address user, uint256 amount);
    event Borrowed(address user, uint256 amount);
    event Repaid(address user, uint256 amount);
    event FlashLoanExecuted(address receiver, uint256 amount, uint256 fee);

    error InsufficientLiquidity();
    error FlashLoanNotRepaid();

    constructor(address _lendingToken) Ownable(msg.sender) {
        lendingToken = IERC20(_lendingToken);
    }

    function supply(uint256 amount) external nonReentrant {
        lendingToken.transferFrom(msg.sender, address(this), amount);
        balances[msg.sender] += amount;
        totalSupplied += amount;
        emit Supplied(msg.sender, amount);
    }

    function borrow(uint256 amount) external nonReentrant {
        require(amount <= totalSupplied / 2, "Exceeds 50% utilization");
        lendingToken.transfer(msg.sender, amount);
        borrows[msg.sender] += amount;
        totalSupplied -= amount;
        emit Borrowed(msg.sender, amount);
    }

    function repay(uint256 amount) external nonReentrant {
        lendingToken.transferFrom(msg.sender, address(this), amount);
        borrows[msg.sender] -= amount;
        totalSupplied += amount;
        emit Repaid(msg.sender, amount);
    }

    // Simple flash loan (receiver must repay + fee in same tx)
    function flashLoan(address receiver, uint256 amount, bytes calldata data) external nonReentrant {
        uint256 balanceBefore = lendingToken.balanceOf(address(this));
        require(amount <= balanceBefore, "Insufficient liquidity");

        uint256 fee = (amount * flashFee) / 10000;
        lendingToken.transfer(receiver, amount);

        // Call receiver contract (must implement flashLoanCallback)
        (bool success,) = receiver.call(abi.encodeWithSignature("flashLoanCallback(uint256,bytes)", amount + fee, data));
        require(success, "Callback failed");

        uint256 balanceAfter = lendingToken.balanceOf(address(this));
        if (balanceAfter < balanceBefore + fee) revert FlashLoanNotRepaid();

        emit FlashLoanExecuted(receiver, amount, fee);
    }

    function withdrawSupply(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        totalSupplied -= amount;
        lendingToken.transfer(msg.sender, amount);
    }
}
