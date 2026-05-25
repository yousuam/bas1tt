# bas1ttimport time
from collections import defaultdict, deque
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 10
COLLAPSE_DROP = 0.80
HISTORY_WINDOWS = 5
BAD_SCORE_THRESHOLD = 3

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect to Base RPC")

    print("Connected to Base")
    print("Detecting bad alpha wallets...\n")

    last_block = w3.eth.block_number

    # token -> activity history
    token_history = defaultdict(
        lambda: deque(maxlen=HISTORY_WINDOWS)
    )

    # token -> wallets involved early
    token_wallets = defaultdict(set)

    # wallet -> bad score
    wallet_scores = defaultdict(int)

    while True:
        try:
            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                from_block = current_block - WINDOW_BLOCKS
                to_block = current_block

                logs = w3.eth.get_logs({
                    "fromBlock": from_block,
                    "toBlock": to_block,
                    "topics": [TRANSFER_TOPIC]
                })

                current_activity = defaultdict(int)
                current_wallets = defaultdict(set)

                for log in logs:

                    token = log["address"]

                    from_addr = decode_address(log["topics"][1])
                    to_addr = decode_address(log["topics"][2])

                    current_activity[token] += 1

                    if from_addr != ZERO:
                        current_wallets[token].add(from_addr)

                    if to_addr != ZERO:
                        current_wallets[token].add(to_addr)

                for token, activity in current_activity.items():

                    history = token_history[token]

                    if len(history) >= 3:

                        avg_previous = sum(history) / len(history)

                        if avg_previous > 0:

                            drop = 1 - (
                                activity / avg_previous
                            )

                            if drop >= COLLAPSE_DROP:

                                for wallet in token_wallets[token]:
                                    wallet_scores[wallet] += 1

                    token_wallets[token].update(
                        current_wallets[token]
                    )

                    history.append(activity)

                print(f"\nBlocks {from_block} → {to_block}")

                for wallet, score in wallet_scores.items():

                    if score >= BAD_SCORE_THRESHOLD:

                        print("💀 Bad Alpha Wallet")
                        print("Wallet:", wallet)
                        print("Failure score:", score)
                        print()

                last_block = current_block

            time.sleep(3)

        except Exception as e:
            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
