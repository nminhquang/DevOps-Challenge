Provide your CLI command here:
for order_id in $(cat transaction-log.txt | jq -r 'select(.symbol == "TSLA") | .order_id'); do curl -L https://example.com/api/${order_id} >> ./output.txt; done