# BASH SCRIPTING

A comprehensive reference guide for Linux commands and Bash scripting.

## Table of Contents
[1. Informational Commands](#1-informational-commands)  
[2. File and Directory Navigation](#2-file-and-directory-navigation)  
[3. File and Directory Management](#3-file-and-directory-management)  
[4. Networking Commands](#4-networking-commands)  
[5. Bash Scripting](#5-bash-scripting)    


```Bash
apt-get install <pckg-name>
apt-get remove <pckg-name>
apt-get update
apt-get upgrade
```

## 1. Informational Commands

```bash
id [-u --user-ID] [-n --user-name]

uname [-s --kernel-name] [-r --kernel-release] [-v --kernel-version] 
      [-n --network-host] [-a --all] [-m --machine] 
      [-p --processor] [-i --hardware] [-o --OS]

who [-a --all] [--boot --login -p -t -u]

df [-h --human-readable] [<dir_path>]

ps [-u root] [-e --running-ps]

sudo netstat -tulpn

top [-n <int>]  # Use shift+[m p n t] to sort table while top is running

date [-r --last-modified] <filename>
     ['+%D %Y %m %d %T] 
     [+%s]
```

## 2. File and Directory Navigation

```bash
pwd                     # Print working directory

ls [-R --recursive-tree-struct] [-l --permissions] [-a --hidden] 
   [-d --dir] [-S --sort-size] [-t --last-modified] [<dir_path>] 

cd [dir_path]           # Change directory

find <parent_dir> [-type f] [-name] <filename> [-exec cmd {}\; ]

which <program>         # Show the full path of commands
```

## 3. File and Directory Management

### Basic Operations

```bash
mkdir <dirname>         # Create directory

rm [-i --confirm-delete] [<filename>] [-r <dirname>] [--force]

rmdir <empty_dirname>   # Remove empty directory

cp [-r] <sourcepath/filename> <destpath/filename>

mv [-r] <sourcepath/filename> <destpath/filename>

chmod [-o -g -u] [-+][xrw] <filename>

touch <filename.ext>    # Create file or update timestamp
```

### Content Manipulation

```bash
cat [-n --line-number] <filename1> [<filename2>, ...] [> | >> <dest_filename>]

less/more <filename>    # View file contents with pagination

head/tail [-n <int, default 10>] <filename>  # View beginning/end of file

echo [-e --escape-char\n\t] <str/var> [> | >> <filename>]
```

### Text Processing

```bash
tr [-c --replace-all-but-pattern] 'regexpattern' 'replacement_pattern'

sed 's/: pattern-to-sub/replacement/' < filename
sed '/^[[:space:]]*$/d'     # Removes blank lines 

wc [-l --lines] [-w --words] [-c --chars] <filename>

sort [-r --reverse] <filename>

uniq <filename>         # Remove duplicate adjacent lines

grep [-i --case-insensitive] [-e --regex | -w --match-full-word]
     [-l --filenames-contains-pattern] [-f --use-patterns-from-file <fn>]
     [-o --stdout-matches] [-v --stdout-lines-w/o-matches] 
     [-n --stdout-matchline-number] [-c --match-count] 
     'pattern' [<filename>]

cut [-c <startchar-endchar-eachline>] 
    [-d 'delimiter' -f<field-number>]
    <filename>

paste [-d 'delimiter'] <file_1> <file_n>
```

### Compression

```bash
tar [-cf --compress -v --verbose <filename.tar>] [-czf <filename.tar.gz>] <source-dirname/filename>
    [-tf --show-contents-tree -v --verbose <filename.tar>]
    [-xf --extract -v --verbose <filename.tar>] [-xzf <filename.tar.gz>] [--directory <dest-dirname/filename>]

zip [-r] <filename.zip> [<] <source-dirname/filename>

unzip [-l --list-files-in-archive] [-o --force-overwrite] <filename.zip> <dest-dirname/filename>

gunzip -f <filename.zip.gz> 
```

## 4. Networking Commands

```bash
hostname [-s --drop-suffix] [-i --ip] 

ifconfig [-s] / netstat [-i]

ping [-c <int>]

curl [-o --save-as <filename>] [-u user:passwd] [--data POST] [--upload-file <filename>] <url>

wget <url>
```

## 5. Bash Scripting

### Variables

```bash
env [-u --unset var_name]     # Display all environment variables

set                           # Display all variables and functions

unset <local_var_name>        # Remove variable

export <local_var_name>       # Make variable available to subprocesses

var=$()                       # Command substitution

declare -a <array>            # Declare an array

array+=($var)                 # Add value to array

read -p 'other values to write:' array   # Read input into variable

echo ${array[@]}              # Print all array elements

echo "${var%rm_suffix}"       # Remove suffix from variable

echo "${var#rm_prefix}"       # Remove prefix from variable

$#  # Number of arguments
$@  # All arguments
$1  # First argument
$_  # Last value returned
$?  # Exit status of a command
```

### Conditionals and Loops

```bash
# If statement
if [[ condition ]]; then 
    statement
elif [[ condition ]]; then 
    statement
else 
    statement
fi 

# For loop
for x in {1..10}; do 
    statement
done

# While loop reading file
while read -r line; do 
    statement
done < file.txt
```

### Functions

```bash
# Define function
myfunction() {
    echo $1
}

# Call function
myfunction "passed_arg"
```