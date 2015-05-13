# free_log_space

Instructions and small copy &amp; paste bash scripts to make space on a drive filled by logs

1. Your log file ate all of your disk?
2. Can't just wipe the log?
3. Can't wait to transfer file over network?
4. You are running Linux?

This will free some disk space without losing data so you get some time to sort it out.

== Step 1:

Run this to MAKE SURE YOU HAVE THE NEEDED TOOLS (most Linux systems do):

    sh -c 'printf %05d 1 | wc -c > /dev/null; base64 --version > /dev/null && stat --version > /dev/null && truncate --version > /dev/null && tail --version > /dev/null && gzip --version > /dev/null && echo && echo "ALL OK" && echo'

If it says "ALL OK", you are good do go.

== Step 2:

STOP THE SERVER that is filling your log and RENAME THE LOG FILE.

It takes only a couple of seconds after starting this script until you can start the server again. Just don't have it on when you start because the script expects your log file is not growing while it's run.

An example:

    /etc/init.d/apache stop
    mv /var/log/apache/access.log /var/log/apache.log.stored

== Step 3:

Start the following script AFTER CHANGING the "/var/log/apache.log.stored" at the end of the following script to point to WHAT YOU RENAMED YOUR LOG AS:

    sh -c 'F=$1;L=1000;I=1;while [ -s $F ] && [ $I -lt 99 ];do P=$F.part$(printf %03d $I);if [ -f $P ]; then echo $P already exists! 1>&2;exit 1;fi;if [ $I -lt 21 ]; then S=$(tail -n $L|gzip|base64 -w0) && truncate -s-$(tail -n $L|wc -c) $F && echo $S|base64 -d > $P.gz;else tail -n ${L}0 $F|gzip > $P.gz && truncate -s-$(tail -n ${L}0 $F|wc -c) $F;fi;I=$(expr $I + 1);done' -- /var/log/apache.log.stored

Wait for a couple of seconds and you should have enough space to start your log-hogging daemons again! You can for example start the daemons again in an another shell session and let this one run it's course.

BE WARNED THOUGH! THERE IS A THEORETICAL CHANCE THAT YOU MIGHT LOSE 1000 OF THE LAST 20000 LINES OF YOUR LOG IF SOMETHING UNEXPECTED BREAKS VERY EARLY IN THE PROCESS.

So what does this do?

1. Takes the last 1000 lines of the log file.
2. Shrinks the lines with gzip to an ENV variable (in memory!).
3. Deletes the last 1000 lines of the log file to free some disk space.
4. Outputs the gzip package to the disk from the ENV variable.
5. Repeats steps 1 to 4 20 times to get some more space.
7. Takes the last 10000 lines of the log file.
8. Shrinks the lines with gzip straight to disk.
9. Deletes the last 10000 lines of the log file.
10. Repeats steps 7 to 9 up to 78 times or until the file is empty.

Here is the code properly indented and with proper variable names so that you know what you are doing:

    sh -c '
        FILE_TO_SPLIT=$1
        LINES_PER_PART=1000
        ITERATION=1
        while [ -s $FILE_TO_SPLIT ] && [ $ITERATION -lt 99 ]
        do
            PART_FILE=$FILE_TO_SPLIT.part$(printf %03d $ITERATION)
            if [ -f $PART_FILE ]
            then
                echo $PART_FILE already exists! 1>&2
                exit 1
            fi
            if [ $ITERATION -lt 21 ]
            then
                STORAGE=$(tail -n $LINES_PER_PART|gzip|base64 -w0) &&
                truncate -s-$(tail -n $LINES_PER_PART|wc -c) $FILE_TO_SPLIT &&
                echo $STORAGE|base64 -d > $PART_FILE.gz
                if [ $? -gt 0 ]
                then
                    echo
                    echo $STORAGE
                    echo
                    echo STORE CODE ABOVE AND READ RECOVERY INSTRUCTIONS!
                    echo
                    exit 1
                fi
            else
                tail -n ${LINES_PER_PART}0 $FILE_TO_SPLIT|gzip > $PART_FILE.gz &&
                truncate -s-$(tail -n ${LINES_PER_PART}0 $FILE_TO_SPLIT|wc -c) $FILE_TO_SPLIT
            fi
            ITERATION=$(expr $ITERATION + 1)
        done
    ' -- /var/log/apache.log.stored

== Recovery:

You are left with the original.file (which might be empty) and a lot of original.file.partNNN.gz files.

If everything went smoothly when running the script, run the following script to get your original file back (after replacing file.log with your file name):

    sh -c 'F=$1; cat $F; ls $F.part*.gz | sort -r | zcat' -- file.log > file.log.original

If you had to stop the script before it finished, you might be in one of the following situations:

1. Only a partial gzip package was written to the disk. You can make sure this is not the case by running the following script:

    sh -c 'ls $1.part*.gz' | sort -r |head -n1| zcat && echo && echo "ALL OK" && echo' -- file.log

If the script does not output "ALL OK", you can remove the partial gzip package:

    sh -c 'ls $1.part*.gz' | sort -r |head -n1| xargs -n1 rm && echo && echo "ALL OK" && echo' -- file.log

2. Gzip was written out properly but the last lines of the original log were not removed properly, leading to duplicate rows:

Use a diff (or antidiff?) for the last 10000 lines of the log and the output of the last gzip.

3. The script was halted during the first 20 in-memory rounds. You might be in trouble.
