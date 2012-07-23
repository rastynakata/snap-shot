snap-shot
=========
#!/bin/bash
# - NAME:
#    snap - output system statistics.
#
# - SYNOPSIS:
#    snap [-h]
#
# - DESCRIPTION:
#    snap outputs overall statistics of the system it is being run on and the
#    date and time it is running, the system 1, 5, 15 minutes load averages, the
#    current % use of the CPU. If there are multiple CPUs, this is the average
#    % use and is any use except idle. All data are based on the output of the
#    top command.
#
# - OPTION(S):
#    -h        In addition of the overall system statistics, snap outputs
#              information about the 5 processes that are using the largest
#              proportion of two valuable system resources: the 5 processes
#              that are currently using the most memory (measured by run-set
#              -size), and the 5 processes that are currently using the
#              highest percentage of the CPU.
#

tmpfile1=$(mktemp /tmp/top.out.XXXX)
tmpfile2=$(mktemp /tmp/procs.XXXX)
tmpfile3=$(mktemp /tmp/procs1.XXXX)
# DELETE "#" from the following line to remove the tmpfile1 after each run.
trap 'rm -f $tmpfile1 $tmpfile2 $tmpfile3' EXIT
# Comment the following line when the "#" is removed from the line above.
#trap 'rm -f $tmpfile2 $tmpfile3' EXIT
trap '' HUP
trap 'echo terminated >&2' TERM
trap 'echo interrupted >&2' INT

# Define a few Color's
BLACK='\e[0;30m'
BLUE='\e[0;34m'
GREEN='\e[0;32m'
CYAN='\e[0;36m'
RED='\e[0;31m'
PURPLE='\e[0;35m'
BROWN='\e[0;33m'
LIGHTGRAY='\e[0;37m'
DARKGRAY='\e[1;30m'
LIGHTBLUE='\e[1;34m'
LIGHTGREEN='\e[1;32m'
LIGHTCYAN='\e[1;36m'
LIGHTRED='\e[1;31m'
LIGHTPURPLE='\e[1;35m'
YELLOW='\e[1;33m'
WHITE='\e[1;37m'
NC='\e[0m'

getinfo() {
    local numProc idlecpu inusecpu name loadavg list
    numProc=$(ps -e | wc -l)
    name=$(whoami)
    if [ "$(uname)" = 'Linux' ]; then
        top -s -d 1 -n 1 -b >$tmpfile1
        idlecpu=$(head -3 $tmpfile1 | tail -1 | tr -s ' ' | cut -d' ' -f8 | cut -d'%' -f1)
        loadavg=$(head -n 1 $tmpfile1 | tr -s ' ' | cut -d',' -f4,5,6 | cut -d':' -f2)
        list=$(tail -n +8 $tmpfile1 | head -n -1 | tr -s ' ' | grep -vE "^([^ ]* ){1}"${name:0:8}" ([^ ]* ){9}top" | tr ' ' '#')
        for i in $list; do
            if [ "$(echo "$i" | cut -c1)" = '#' ]; then
                echo "${i#?}" | cut -d'#' -f1,2,5,9,11,12
            else
                echo "$i" | cut -d'#' -f1,2,5,9,11,12
            fi
        done >$tmpfile2
    elif [ "$(uname)" = 'HP-UX' ]; then
        top -f $tmpfile1 -s 1 -d 1 -n "$numProc"
        tail +14 $tmpfile1 | tr -s ' ' |  tr ' ' '#' |grep -vE "^#([^#]*#){3}"${name:0:8}"#([^#]*#){8}top" | cut -d"#" -f4,5,9,10,13,14 >$tmpfile2
        idlecpu=$(head -9 $tmpfile1 | tail -1 | tr -d '%' | tr -s ' ' | cut -d" " -f6)
        loadavg=$(head -2 $tmpfile1 | tr -s ' ' | cut -d' ' -f3,4,5 | tail -n 1)
    else
        echo "$(basename $0): ERROR: $(basename $0) does not work on your system" >&2
    fi
    # echo "$idlecpu"
    inusecpu=$(echo "100-$idlecpu" | bc)
    echo -e ${WHITE}"Generating summary statistics on $(hostname) at $(date)"
    echo -e ${WHITE}"Load averages:" ${LIGHTGREEN}"$loadavg"
    echo  -e ${WHITE}"The cpu is currently" ${LIGHTRED}"$inusecpu%" ${WHITE}"in use."
    echo -ne ${YELLOW}
    echo  -e "$(echo "scale=2; $(free -k -o | head -n 2 | tail -n 1 | tr -s " " | cut -d " " -f 3)*100/\
              $(free -k -o | head -n 2 | tail -n 1 | tr -s " " | cut -d " " -f2)" | bc)%" ${WHITE}"memory in use."
    echo -ne ${LIGHTRED}
    echo -e "$(free -m -o | grep Mem | tr -s ' ' | cut -d ' ' -f4)mb" ${WHITE}"free out of"\
             ${LIGHTCYAN}"$(free -m -o | grep Mem | tr -s ' ' | cut -d ' ' -f2)mb" ${WHITE}"memory."
    echo -e ${BROWN}"$(ps -e | wc -l)" ${WHITE}"processes on the system." ${BROWN}"$(who | cut -d" " -f1 | sort | uniq | wc -l)" ${WHITE}"unique users."
    echo -ne ${LIGHTCYAN}
    echo "Summary:"
    perl -e 'print "#" x 70, "\n"'
    echo -ne ${LIGHTPURPLE}
    who | cut -d" " -f1 | sort | uniq -c | sort -nr
    echo -ne ${LIGHTCYAN}
    perl -e 'print "#" x 70, "\n"'
    echo -ne ${NC}
}

if [ $# -gt 1 ]; then
    echo "$(basename $0): ERROR: $(basename $0) takes 1 or no argument" >&2
    echo "$(basename $0): usage: $(basename $0) [-h]" >&2
    exit 1
fi

if [ $# -eq 0 ]; then
    getinfo
elif [ $# -eq 1 ]; then
    if [ "$1" = '-h' ]; then
        getinfo
        pid=$(cut -d"#" -f1 $tmpfile2)
        for id in $pid; do
            if grep -iq "^$id#[^#]*#[^#]*m#" $tmpfile2; then
                size=$(grep "^$id#" $tmpfile2 | cut -d"#" -f4)
                realsize=$(echo "${size%?}")
                rss=$(echo "$realsize*1024" | bc)
            else
                rss=$(grep "^$id#" $tmpfile2 | cut -d"#" -f4 | cut -dm -f1)
            fi
            user=$(grep "^$id#" $tmpfile2 | cut -d"#" -f2)
            state=$(grep "^$id#" $tmpfile2 | cut -d"#" -f5)
            cpu=$(grep "^$id#" $tmpfile2 | cut -d"#" -f3)
            realcpu=$(echo "scale=2; $cpu*100/100" | bc)
            name=$(grep "^$id#" $tmpfile2 | cut -d"#" -f6)
            echo "$id#$user#$rss""K#$state#$realcpu#$name"
        done >$tmpfile3
        echo "here are the five biggest cpu hogs:"
        format='%-10.10s %-10.10s %-10.10s %-10.10s %-10.10s %-s'
        printf "${format}\n" PID USER RSS STATE CPU% NAME
        for i in $(sort -t# -r -n -k5  $tmpfile3 | head -5); do
            echo "$i" | awk -F'#' -v OFS='' -v format="$format" '{printf format "\n", $1, $2, $3, $4, $5, $6}'
        done
        echo " "
        echo "here are the five biggest memory hogs:"
        printf "${format}\n" PID USER RSS STATE CPU% NAME
        for n in $(sort -t# -r -n +2 -3 $tmpfile3 | head -5); do
            echo "$n" | awk -F'#' -v OFS='' -v format="$format" '{printf format "\n", $1, $2, $3, $4, $5, $6}'
        done
        exit 0
    else
        echo "$(basename $0): usage: $(basename $0) [-h]" >&2
        exit 1
    fi
fi
