# migrate-redis (for Ubuntu only )
Bash script for redis migration

### live migrate
```
while true; do exec redis-cli [source redis] monitor | grep --line-buffered -vi 'get\|exist\|ping' | grep --line-buffered -i '[key patterns]' | sed -u 's/.*]//' | redis-cli [target redis] ;done;
```
Note:
1. use `> /dev/null` to discard output , or remove it to see command status
2. for `mac terminal` change `sed -u` to `sed -l`

### migrate existing keys (ref: https://elliotchance.medium.com/migrating-data-between-redis-servers-without-downtime-429e4c8048e6 )
```
#clear up dump log
echo "" > redis_dump_keys.log

#scan by pattern
for KEY in $($OLD --scan | grep $PATTERN); do
    $OLD --raw DUMP "$KEY" | head -c-1 > /tmp/dump
    TTL=$($OLD --raw TTL "$KEY")
    #capture key ttl
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
            #TTL in restore is in miliseconds so we need to add 000
            cat /tmp/dump | $NEW -x RESTORE "$KEY" "$TTL""000"
            ;;
    esac
    #print log
    echo "$KEY (TTL = $TTL)"
    #dump log to file for analysis
    echo "$KEY (TTL = $TTL)" >> redis_dump_keys.log
done

echo "Dump log created! Please check redis_dump_keys.log to validate"
```

1. copy script to file.sh
2. chmod 777 file.sh
3. Run script `OLD="[source redis]" NEW="[target redis]" PATTERN="[grep pattern]" ./file.sh`


Example
```
OLD="redis-cli -p 6379" NEW="redis-cli -p 6380" PATTERN="user_" ./file.sh
```
