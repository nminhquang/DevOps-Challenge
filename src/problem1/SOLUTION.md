Because transaction-log.txt file is in json format so we can use jq command to get the order_id before make get request:

for order_id in $(cat transaction-log.txt | jq -r 'select(.symbol == "TSLA") | .order_id'); do curl -L https://example.com/api/${order_id} >> ./output.txt; done
