# super-hosts
Super hosts file amalgamated from several sources to suit the needs of my network. Contains over 68,000 unique entries. The bulk of the hosts come from Steven Black's https://github.com/StevenBlack/hosts, which was also used to generate this host file. I put it here so that my DD-WRT enabled router can pull the hosts file on boot.
Below is the script I use on my router (in the Administration/Commands page). I got it from a ddwrt forum, but no longer have the source/author info to give proper credit.I leave the top (#commented) line when pasting into my router just so I have the instructions handy when sending it to other people to use.
You'll probably want to view this in raw format for copying. I'll pretty it up later.

# --- COPY THE TEXT BELOW TO DD-WRT / ADMINISTRATION / COMMANDS then click SAVE FIREWALL ---
BH_SCRIPT="/tmp/blocking_hosts.sh"
BH_WHITELIST="/tmp/blocking_hosts.whitelist"
logger "Download blocking hosts file and restart dnsmasq ..."
# Create whitelist. The whitelist entries will be removed from the
# hosts files, i.e. blacklist files.
cat > "$BH_WHITELIST" <<EOF
localhost\\.localdomain
local
invalid
whitelist-example\\.com
.*\\.whitelist-example\\.com
EOF
# Create download script.
cat > "$BH_SCRIPT" <<EOF
#!/bin/sh
# Function: clean_hosts_file [file ...]
clean_hosts_file() {
  # The sed script cleans up the file.
  # The awk script groups the hosts by ten items.
  sed -e '/^127.0.0.1/b replace;
          /^0.0.0.0/b replace;
          :drop;
            d; b;
          :replace;
            s/^0.0.0.0[[:space:]]*//;
            s/^127.0.0.1[[:space:]]*//;
            s/[[:space:]]*#.*\$//;
            s/[[:space:]]*\$//;
            s/[[:space:]][[:space:]]*/ /;
            /^localhost\$/b drop;
            /^[[:space:]]*\$/b drop;' \$* | \\
  awk 'BEGIN {
         # Read whitelist file.
         n_whitelist = 0
         while ( getline < "$BH_WHITELIST" ) {
           if ( \$0 == "" ) {
             break
           }
           else {
             a_whitelist[++n_whitelist] = \$0
           }
         }
         close("$BH_WHITELIST")
         # Setup record sparator.
         RS=" +"
         c = 0
       }
       {
         for ( n = 1; \$n != ""; n++ ) {
           # Check whitelist.
           whitelist_flag = 0
           for ( w = 1; w <= n_whitelist; w++ ) {
             if ( \$n ~ ( "^" a_whitelist[w] "\$" ) ) {
               whitelist_flag = 1
               break
             }
           }
           if ( whitelist_flag == 0 ) {
             hosts[++c] = \$n
             if ( c == 10 ) {
               s_hosts = "0.0.0.0"
               for ( i = 1; i <= c; i++ ) {
                 s_hosts = s_hosts " " hosts[i]
               }
               print s_hosts
               c = 0
             }
           }
         }
       }
       END {
        if ( c > 0 ) {
           s_hosts = "0.0.0.0"
           for ( i = 1; i <= c; i++ ) {
             s_hosts = s_hosts = s_hosts " " hosts[i]
           }
           print s_hosts
         }
       }'
}
# Function: wait_for_connection
wait_for_connection() {
  # Wait for an Internet connection.
  # This possibly could take a long time.
  while :; do
    ping -c 1 -w 10 www.freebsd.org > /dev/null 2>&1 && break
    sleep 10
  done
}
# Set lock file.
LOCK_FILE="/tmp/blocking_hosts.lock"
# Check lock file.
if [ ! -f "\$LOCK_FILE" ]; then
  sleep \$((\$\$ % 5 + 5))
  [ -f "\$LOCK_FILE" ] && exit 0
  echo \$\$ > "\$LOCK_FILE"
  # Start downloading files.
  HOSTS_FILE_NUMBER=1
  [ -d "/tmp/blocking_hosts" ] || mkdir "/tmp/blocking_hosts"
  for URL in "https://raw.githubusercontent.com/hoshsadiq/adblock-nocoin-list/master/hosts.txt" \\
             "https://raw.githubusercontent.com/marinegundoctor/super-hosts/master/hosts"; do
    HOSTS_FILE="/tmp/blocking_hosts/hosts\`printf '%02d' \$HOSTS_FILE_NUMBER\`"
    logger "Downloading \$URL ..."
    REPEAT=1
    while :; do
      # Wait for internet connection.
      wait_for_connection
      START_TIME=\`date +%s\`
      # Create process to download a hosts file.
      wget -O - "\$URL" 2> /dev/null > "\${HOSTS_FILE}.tmp" &
      WGET_PID=\$!
      WAIT_TIME=\$((\$REPEAT * 10 + 20))
      # Create timeout process.
      ( sleep \$WAIT_TIME; kill -TERM \$WGET_PID ) &
      TIMEOUT_PID=\$!
      wait \$WGET_PID
      CURRENT_RC=\$?
      kill -KILL \$TIMEOUT_PID
      STOP_TIME=\`date +%s\`
      if [ \$CURRENT_RC = 0 ]; then
        clean_hosts_file "\${HOSTS_FILE}.tmp" > "\$HOSTS_FILE"
        rm "\${HOSTS_FILE}.tmp"
        break
      fi
      # In the case of an error: wait the remaining time.
      TIME_SPAN=\$((\$STOP_TIME - \$START_TIME))
      WAIT_TIME=\$((\$WAIT_TIME - \$TIME_SPAN))
      [ \$WAIT_TIME -gt 0 ] && sleep \$WAIT_TIME
      # Increase the number of repeats.
      REPEAT=\$((\$REPEAT + 1))
      [ \$REPEAT = 4 ] && break
    done
    HOSTS_FILE_NUMBER=\$((\$HOSTS_FILE_NUMBER + 1))
  done
  # Inspect downloaded hosts files.
  ANY_FILE_OK=1
  DNSMASQ_PARAM=""
  for HOSTS_FILE in /tmp/blocking_hosts/hosts[0-9][0-9]; do
    if [ -s "\$HOSTS_FILE" ]; then
      ANY_FILE_OK=0
      DNSMASQ_PARAM=\${DNSMASQ_PARAM:+\$DNSMASQ_PARAM }"--addn-hosts=\$HOSTS_FILE"
    else
      rm "\$HOSTS_FILE"
    fi
  done
  if [ \$ANY_FILE_OK = 0 ]; then
    logger "Restarting dnsmasq with additional hosts file(s) ..."
    killall -TERM dnsmasq
    dnsmasq --conf-file=/tmp/dnsmasq.conf \$DNSMASQ_PARAM &
  fi
  rm "\$LOCK_FILE"
fi
EOF
# Make it executeable.
chmod 755 "$BH_SCRIPT"
# Add crontab entry.
grep -q "$BH_SCRIPT" /tmp/crontab || echo "$(($$ % 60)) 3 * * * root $BH_SCRIPT" >>/tmp/crontab
# Execute script in background.
sh "$BH_SCRIPT" &
