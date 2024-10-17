Searching for a file by name or extension
Finding files/directories with  specific name, relative to pwd:

$ find . -name "myFile.txt"
./myFile.txt
Finding files/directories with a specific extension, use a wildcard:

$ find . -name "*.txt"
./myFile.txt
./myFile2.txt
Finding files/directories to match one of many extensions, use the or flag:

$ find . -name "*.txt" -o -name "*.sh"
Finding files/directories which name begin with abc and end with one alpha character that a one digit follows:

$ find . -name "abc[a-z][0-9]"
Finding all files/directories located in a specific directory

$ find /opt
Searching for files only (not directories), use -type f:

find /opt -type f
Searching for directories only (not regular files), use -type d:

find /opt -type d
Finding files by type
To find files, use the -type f flag

$ find . -type f
Findind directories, use the -type d flag

$ find . -type d
Finding block devices, use the -type b flag

$ find /dev -type b
Finding symlinks, use the -type l flag

$ find . -type l
Executing commands against a found file
Sometimes we will need to run commands against a lot of files. This can be done with  xargs.

find . -type d -print | xargs -r chmod 770
The above command will recursively find all directories (-type d) relative to . (which is our current working directory), and execute chmod 770 on them. The -r option specifies to xargs to not run chmod if find did not find any files.

If our files names or directories have a space character in them, this command may choke; a solution is to use the following

find . -type d -print0 | xargs -r -0 chmod 770
In the above example, the -print0 and -0 flags specify that the file names will be separated using a null byte, and allows the use of special characters, like spaces, in the file names. This is a GNU extension, and may not work in other versions of find and xargs.

Skipping the xargs command and let find call the subprocess itself is the prefered to do it:

find . -type d -exec chmod 770 {} \;
Here, the {} is a placeholder that indicate we want to use the file name at that point. find will execute chmod on each file individually.

You can alternatively pass all file names to a single call of chmod, by using

find . -type d -exec chmod 770 {} +
This is also the behaviour of the above xargs snippets. (To call on each file individually, we can use xargs -n1).

Letting bash to loop over the list of filenames to find outputs is a third option:

find . -type d | while read -r d; do chmod 770 "$d"; done
Syntactically, it's the most clunky, but have convenience  when we want to run multiple commands on each found file. However, this is unsafe in the face of file names with odd names.

find . -type f | while read -r d; do mv "$d" "${d// /_}"; done
which will replace all spaces in file names with underscores.(This example also won't work if there are spaces in leading directory names.)

The problem with the above is that while read -r expects one entry per line, but file names can contain newlines (and also, read -r will lose any trailing whitespace). We can fix this by turning things around:
z
find . -type d -exec bash -c 'for f; do mv "$f" "${f// /_}"; done' _ {} +
This way, the -exec receives the file names in a form which is completely correct and portable; the bash -c receives them as a number of arguments, which will be found in $@, correctly quoted etc. (The script will need to handle these names correctly, of course; every variable which contains a file name needs to be in double quotes.)

The mysterious _ is necessary because the first argument to bash -c 'script' is used to populate $0.

Finding file by access / modification time
On an ext filesystem, each file has a stored Access, Modification, and (Status) Change time associated with it - to view this information we can use stat myFile.txt; using flags within find, you can search for files that were modified within a certain time range.

Finding files that have been modified within the last 2 hours:

$ find . -mmin -120
Finding files that have not been modified within the last 2 hours:

$ find . -mmin +120
The above example are searching only on the modified time - to search on access times, or changed times, use a, or c accordingly.

$ find . -amin -120
$ find . -cmin +120
General format:

-mmin n : File was modified n minutes ago
-mmin -n : File was modified less than n minutes ago
-mmin +n : File was modified more than n minutes ago

Finding files that have been modified within the last 2 days:

find . -mtime -2
Finding files that have not been modified within the last 2 days

find . -mtime +2
Using -atime and -ctime for access time and status change time respectively.

General format:

-mtime n : File was modified nx24 hours ago
-mtime -n : File was modified less than nx24 hours ago
-mtime +n : File was modified more than nx24 hours ago

Finding files modified in a range of dates, from 2007-06-07 to 2007-06-08:

find . -type f -newermt 2007-06-07 ! -newermt 2007-06-08  
Finding files accessed in a range of timestamps (using files as timestamp), from 1 hour ago to 10 minutes ago:

touch -t $(date -d '1 HOUR AGO' +%Y%m%d%H%M.%S) start_date
touch -t $(date -d '10 MINUTE AGO' +%Y%m%d%H%M.%S) end_date
timeout 10 find "$LOCAL_FOLDER" -newerat "start_date" ! -newerat "end_date" -print  
General format:

-newerXY reference : Compares the timestamp of the current file with reference. XY could have one of the following values: at (access time), mt (modification time), ct (change time) and more. reference is the name of a file whe want to compare the timestamp specified (access, modification, change) or a string describing an absolute time.

Finding files by specific extension
To find all the files of a certain extension within the current path you can use the following find syntax. It works by making use of bash's built-in glob construct to match all the names having the .extension.

find /directory/to/search -maxdepth 1 -type f -name "*.extension"
Findinh all files of type .txt from the current directory alone, do

find . -maxdepth 1 -type f -name "*.txt"
Finding files according to size
Find files larger than 15MB:

find -type f -size +15M
Finding files less than 12KB:

find -type f -size -12k
Finding files exactly of 12KB size:

find -type f -size 12k
Or

find -type f -size 12288c
Or

find -type f -size 24b
Or

find -type f -size 24
General format:

find [options] -size n[cwbkMG]
Finding files of n-block size, where +n means more than n-block, -n means less than n-block and n (without any sign) means exactly n-block

Block size:

c: bytes
w: 2 bytes
b: 512 bytes (default)
k: 1 KB
M: 1 MB
G: 1 GB
Filter the path
The -path parameter allows to specify a pattern to match the path of the result. The pattern can match also the name itself.

Finding only files containing log anywhere in their path (folder or name):

find . -type f -path '*log*'
Finding only files within a folder called log (on any level):

find . -type f -path '*/log/*'
Finding only files within a folder called log or data:

find . -type f -path '*/log/*' -o -path '*/data/*'
Finding all files except the ones contained in a folder called bin:

find . -type f -not -path '*/bin/*'
Finding all file all files except the ones contained in a folder called bin or log files:

find . -type f -not -path '*log' -not -path '*/bin/*'