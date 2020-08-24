# Building parquet-mr from windows 10, using WSL

The other day, using [NetBeans](https://netbeans.apache.org/) I tried building `mvn clean build` the parquet project (<https://github.com/apache/parquet-mr>) on a Windows 10 machine, but I kept getting this error message:

```none
Failed to execute goal org.apache.thrift:thrift-maven-plugin:0.10.0:compile (thrift-sources) on project parquet-format-structures: thrift did not exit cleanly. Review output for more information. -> [Help 1]

To see the full stack trace of the errors, re-run Maven with the -e switch.
Re-run Maven using the -X switch to enable full debug logging.

For more information about the errors and possible solutions, please read the following articles:
[Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException

After correcting the problems, you can resume the build with the command
  mvn <args> -rf :parquet-format-structures
```

I searched online, but couldn't find any solutions. I went back to the "parquet mr" project on github, and read the README carefully. It turns out I missed a dependency section saying that Apache Thrift has to be installed, that helped clarify my error message. I looked into installing thrift on Windows [http://thrift.apache.org/docs/install/windows](http://thrift.apache.org/docs/install/windows), I didn't like what I saw. Too many steps, and there is a likelihood it wouldn't work. I instead decided to use a linux environment on windows, WSL seemed the most appropriate, given it's lightweight, and I can easily access my windows files.

## WSL to the rescue

I used the Ubuntu 20.04.1 LTS WSL (Windows Subsystem for Linux).

### Step 0 - update/upgrade

```sh
# Update repo
sudo apt update
# Upgrade pacakges if any
sudo apt upgrade
```

### Step 1 - Install Thrift dependencies - Ubuntu/Debian

```sh
# https://thrift.apache.org/docs/install/debian
sudo apt-get install automake bison flex g++ git libboost-all-dev libevent-dev libssl-dev libtool make pkg-config
```

### Step 2 - Install Thrift

```sh
wget -nv http://archive.apache.org/dist/thrift/0.12.0/thrift-0.12.0.tar.gz
tar xzf thrift-0.12.0.tar.gz
cd thrift-0.12.0
chmod +x ./configure
./configure --disable-libs
sudo make install
```

### Step 3 - Clone project

```sh
git clone https://github.com/apache/parquet-mr
cd parquet-mr
```

### Step 4 - Build

```sh
LC_ALL=C mvn clean install
```

![A few monets later - Maybe like 20 mins or so](..\..\res\img\much-much-later.gif)

```none
[ERROR] Failed to execute goal on project parquet-thrift: Could not resolve dependencies for project org.apache.parquet:parquet-thrift:jar:1.12.0-SNAPSHOT: Failed to collect dependencies at com.twitter.elephantbird:elephant-bird-pig:jar:4.4 -> com.twitter.elephantbird:elephant-bird-core:jar:4.4 -> com.hadoop.gplcompression:hadoop-lzo:jar:0.4.16: Failed to read artifact descriptor for com.hadoop.gplcompression:hadoop-lzo:jar:0.4.16: Could not transfer artifact com.hadoop.gplcompression:hadoop-lzo:pom:0.4.16 from/to twitter (http://maven.twttr.com): Transfer failed for http://maven.twttr.com/com/hadoop/gplcompression/hadoop-lzo/0.4.16/hadoop-lzo-0.4.16.pom: Connect to maven.twttr.com:80 [maven.twttr.com/1.2.3.1] failed: Connection refused -> [Help 1]
[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/DependencyResolutionException
[ERROR]
[ERROR] After correcting the problems, you can resume the build with the command
[ERROR]   mvn <args> -rf :parquet-thrift
```

The build failed. I did a quick search, but couldn't find immediate response. But since this build takes a couple minutes before failing, I decided to re-run it with test disabled, maybe these might be test dependencies.

```sh
LC_ALL=C mvn clean install -U -DskipTests=true
```

![Oh no!!! Build failed, again](..\..\res\img\oh-no-bounce.gif)

```none
Too many files with unapproved license: 801 See RAT report in: /home/ubuser/workspace/parquet-mr/target/rat.txt
```

I get a different error this time around... Argh!!! Searching around, I found a flag to disable `RAT(http://creadur.apache.org/rat/)` check.

```sh
LC_ALL=C mvn clean install -U -DskipTests=true -Drat.skip=true package
```

![Not again!!! The old error is back](..\..\res\img\i-cant-believe-it-90-day-fiance.gif)

```none
[ERROR] Failed to execute goal on project parquet-thrift: Could not resolve dependencies for project org.apache.parquet:parquet-thrift:jar:1.12.0-SNAPSHOT: Failed to collect dependencies at com.twitter.elephantbird:elephant-bird-pig:jar:4.4 -> com.twitter.elephantbird:elephant-bird-core:jar:4.4 -> com.hadoop.gplcompression:hadoop-lzo:jar:0.4.16: Failed to read artifact descriptor for com.hadoop.gplcompression:hadoop-lzo:jar:0.4.16: Could not transfer artifact com.hadoop.gplcompression:hadoop-lzo:pom:0.4.16 from/to twitter (http://maven.twttr.com): Transfer failed for http://maven.twttr.com/com/hadoop/gplcompression/hadoop-lzo/0.4.16/hadoop-lzo-0.4.16.pom: Connect to maven.twttr.com:80 [maven.twttr.com/1.2.3.1] failed: Connection refused -> [Help 1]
```

The old error came back. I went back and searched, then came up to this post <https://stackoverflow.com/questions/16729023/maven-build-issue-connection-to-repository-refused>, it suggested I do a telnet to the endpoint `telnet maven.twttr.com 80`. It turns out that for some reason I didn't figure out, the machine I was using couldn't reach that endpoint. I ended up trying a different machine, then repeated steps 0 to 4, and things worked, the parquet project finished compilation after 30 mins.

### Step 5 - Compile local (on the new machine)

Trying to invoke the parquet-tools jar `java -jar parquet-tools-1.12.0-SNAPSHOT.jar`, I get errors saying it couldn't find some class path (*I didn't store that error message*). This error was hit because I didn't build parquet for local (*it seems the default compilation is for hadoop environment*), as a result any reference to classes within the `org.apache.hadoop.fs.*` namespace throws exceptions.

```sh
# Build for local, skipping tests (takes 5 mins, instead of 30)
mvn clean package -Plocal  -DskipTests=true
```

Bingo, now I have officially built the parquet project.

### Step 6 - Use parquet-tools to access files on windows

```sh
target_file="/mnt/<DRIVE>/<PATH>/<SUB-PATH>/<FILE-NAME>.parquet"
out_dir="/mnt/<DRIVE>/<PATH>/<OUT-SUB-PATH>/"

# Try a few commands - Read more, https://github.com/apache/parquet-mr/blob/master/parquet-tools/src/main/java/org/apache/parquet/tools/command/Registry.java
java -jar parquet-tools-1.12.0-SNAPSHOT.jar schema  $target_file >> $out_dir"schema.txt"
java -jar parquet-tools-1.12.0-SNAPSHOT.jar meta    $target_file >> $out_dir"meta.txt"
java -jar parquet-tools-1.12.0-SNAPSHOT.jar dump    $target_file >> $out_dir"dump.txt"
java -jar parquet-tools-1.12.0-SNAPSHOT.jar merge   $target_file >> $out_dir"merge.txt"
```

### Extras

When I started getting the class path exceptions, I thought, maybe all I got to do is copy all the resources into a common folder, then call my jar... That didn't work

```sh
mkdir hot-stuff

# Move all jar files
find parquet-mr/ \( -type f -a -name \*.jar \) -exec cp {} hot-stuff \;
# Move all class files, starting at the root package name "org"
find parquet-mr/ \( -type d -a -wholename "*target/classes/org" \) -exec cp -r {} hot-stuff \;

# Explanation of the find query
# -type: f=files, d=directory
# - -a: Logical AND
```
