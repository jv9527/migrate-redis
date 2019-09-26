# migrate-redis
Bash script for redis migration

### live migrate
```
while true; do exec redis-cli [source redis] monitor | grep --line-buffered -vi 'get\|exist\|ping' | grep --line-buffered -i '[key patterns]' | sed -u 's/.*]//' | redis-cli [target redis] ;done;
```
Note:
1. use `> /dev/null` to discard output , or remove it to see command status
2. for `mac terminal` change `sed -u` to `sed -l`

### migrate existing keys
```
for KEY in $($OLD --scan | grep $PATTERN); do
    $OLD --raw DUMP "$KEY" | head -c-1 > /tmp/dump
    TTL=$($OLD --raw TTL "$KEY")
    case $TTL in
        -2)
            $NEW DEL "$KEY"
            ;;
        -1)
            $NEW DEL "$KEY"
            cat /tmp/dump | $NEW -x RESTORE "$KEY" 0
            ;;
        *)
            $NEW DEL "$KEY"
            cat /tmp/dump | $NEW -x RESTORE "$KEY" "$TTL"
            ;;
    esac
    echo "$KEY (TTL = $TTL)"
done
```

1. copy script to file.sh
2. chmod 777 file.sh
3. Run script `OLD="[source redis]" NEW="[target redis]" PATTERN="[grep pattern]" ./file.sh`
