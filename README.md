import os
from web3 import Web3

RPC_URL = os.getenv("RPC_URL")
PRIVATE_KEY = os.getenv("PRIVATE_KEY")
CONTRACT_ADDRESS = os.getenv("CONTRACT_ADDRESS")

if not all([RPC_URL, PRIVATE_KEY, CONTRACT_ADDRESS]):
    raise RuntimeError(
        "Missing environment variables."
    )

w3 = Web3(Web3.HTTPProvider(RPC_URL))

if not w3.is_connected():
    raise ConnectionError("RPC connection failed.")

account = w3.eth.account.from_key(PRIVATE_KEY)
contract_address = Web3.to_checksum_address(
    CONTRACT_ADDRESS
)

ABI = [
    {
        "inputs": [
            {
                "internalType": "uint256",
                "name": "lockDuration",
                "type": "uint256"
            }
        ],
        "name": "deposit",
        "outputs": [],
        "stateMutability": "payable",
        "type": "function"
    },
    {
        "inputs": [],
        "name": "timeRemaining",
        "outputs": [
            {
                "internalType": "uint256",
                "name": "",
                "type": "uint256"
            }
        ],
        "stateMutability": "view",
        "type": "function"
    }
]

vault = w3.eth.contract(
    address=contract_address,
    abi=ABI
)

def lock_eth(amount_eth, duration_seconds):
    value = w3.to_wei(amount_eth, "ether")
    nonce = w3.eth.get_transaction_count(
        account.address
    )

    tx = vault.functions.deposit(
        duration_seconds
    ).build_transaction({
        "from": account.address,
        "value": value,
        "nonce": nonce,
        "chainId": w3.eth.chain_id,
        "gas": 150000,
        "maxFeePerGas": w3.to_wei(30, "gwei"),
        "maxPriorityFeePerGas": w3.to_wei(
            1, "gwei"
        )
    })

    signed = account.sign_transaction(tx)

    tx_hash = w3.eth.send_raw_transaction(
        signed.raw_transaction
    )

    print("Transaction:", tx_hash.hex())

    receipt = w3.eth.wait_for_transaction_receipt(
        tx_hash
    )

    print("Confirmed:", receipt.blockNumber)


def check_lock():
    seconds = vault.functions.timeRemaining().call()

    hours = seconds // 3600
    minutes = (seconds % 3600) // 60

    print(
        f"Remaining: {hours}h {minutes}m"
    )


# Example:
# lock_eth(0.01, 86400)

check_lock()
