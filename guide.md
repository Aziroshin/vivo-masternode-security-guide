## Guide Style & Further Information

Commands and other important, **literal** pieces of system information are written in *italic* script. Examples: *user* when referring to a system user called "user", or *ls -la* when referring to the command "ls -la", as literally typed into the terminal window.

In case you are curious about any of the commands provided in this guide beyond the explanations it provides, you can always prefix a basic command with "man" and run that in order to learn more about it, or run it with the "--help" argument to get a basic rundown. Example: *ls -la*, which would give you a somewhat more detailed listing of a directory's contents. If you want to learn more about it, you can either type *man ls* (without the *-la* part), or *ls --help*.

#### Assumptions about your server (eg. VPS) Setup

Operating system wise, the guide is written around you using *Ubuntu 16.04*.

Regarding our text editing needs on the server side, we will assume you'll be using *nano*. If you decide to use something else, such as *vi*, substitute accordingly. You can choose your default text editor with the following command, which will come in handy with certain utilities that open a text editor for you, such as *crontab -e* or *visudo*: *sudo update-alternatives --config editor*.

Some basic things you should know about *nano*: You save a file with Ctrl-O and quit the editor with Ctrl-X. You can go into text finding mode by hitting Ctrl-W.

## Setting up the Basic Environment

On conventional operating systems, it is not recommended to use system users with administrative privileges (such as *root* in the case of Linux) to run software that doesn't need those privileges to function properly. Thus, we are going to set up two system users: One to log in to, manage our masternodes and run wallet operations (such as those you can carry out with *vivo-cli* or *vivo-tx*). It's possible you already have such a user without administrative privileges, in which case you won't have to create another user of that sort.

Whatever is the case, we will refer to that user as *user* from here on out. Feel free to use a different name as you see fit and substitute that name whenever this guide references *user*. In fact, in the spirit of security by obscurity (which is a weak form of security, but it can help nonetheless), it might be a good idea if not everyone running a VIVO masternode used the same name for either this user, or the daemon user described below for that matter.

The second user will run the wallet daemon (*vivod*), which is the background process that connects to the blockchain network and does all the things related to that, including masternode operations. The reason we're entertaining such a separation is because we want to cage in that process further, which we'll be doing by restricting what that user can do to a degree that wouldn't be feasible for the other user.

As the daemon is the most vulnerable attack vector in the entire VIVO software suite (or any other crypto software with a similar architecture), such special attention makes sense, as it would restrict what someone that managed to gain enough control over the daemon process to compromise its environment can do. The less damage an attacker can inflict in such an event, the better. For the sake of this guide, we will call this user *vivo-daemon*. As with *user*, feel free to deviate at your discretion.

To set these users up, we are going to go through the following steps:
1. Open a SSH session as the user *root* using your preferred SSH tooling.
2. To create *user*, we will run the following command: *sudo adduser --shell=/bin/bash --gecos "" user*. Fill in the password as prompted. Be advised that the input fields will not show any censoring characters such as stars or dots, but simply remain blank as you're successfully writing in your desired password. In case you exit the program before you can successfully set a password, you can still do so using the following command: *passwd user*.
    * Note: The *--gecos ""* parameter suppresses the program from asking various additional questions, such as full name or phone number. If you'd like to file these values in for whatever reason, leave it out, which would translate to the command looking like this: *sudo adduser --shell=/bin/bash user*.
3. To create *vivo-daemon*, the command will be this: *sudo adduser --shell=/bin/false --gecos "" --disabled-login vivo-daemon*. The *--shell=/bin/false* part is at the heart of why we have this user to begin with: An attacker won't be able to obtain shell access, even if they manage to break in on a level they would otherwise be able to obtain it, which may limit what they can do, depending on the anatomy of the attack.
4. We will want to run *root* level commands as *user*. We will be doing that through *sudo*, which will prompt for a password in order for these commands to execute. We will also want *user* to be able to run *vivod* as *vivo-daemon* without being prompted for a password (as to be able to automate it. Remember: Our security approach assumes that *user* is more trustworthy than *vivo-daemon*). The following set of steps and notes will explain various ways of doing this.
    1. According to the default configuration of Ubuntu 16.04, in order for an ordinary system user, such as *user*, to be able to run commands as root (or any other user for that matter), they are to be added to the system group *sudo*. This is done by running the following command: *usermod -a -G sudo user*, whereas *-G sudo* specifies the group to add *user* to and *-a* tells it to do so without removing *user* from other groups (with "a" standing for "append", instead of "set", if you will).
    2. To configure *user* to be able to run commands as *vivo-daemon* using *sudo*, we will add a file at */etc/sudoers.d/99-vivo*. The files in that directory get included in the main configuration file of *sudo*, */etc/sudoers*. However, we will not directly edit the file, but use *visudo*, which does two things: Start your configured favourite editor with either the default file, */etc/sudoers*, or a specified file opened **and** verify whether there are any issues with the changes you've made whenever you attempt to save them, making sure that the file is not left in a derelict state. For our needs, run the command *visudo -f /etc/sudoers.d/vivo* to start editing.
    3. Put the following into the appearing text field (don't forget to substitute the path of *vivod* and the user names according to your setup):

            user ALL=(vivo-daemon:vivo-daemon) NOPASSWD: /usr/local/bin/vivod

    4. Save & close.

5. Try logging in as *user* instead of *root* to see whether it works. The rest of the guide will assume you're logged in as *user* unless specified otherwise.
6. Run *sudo -la /root* and type in your password once it asks. This can have multiple outcomes (**Do not proceed with the guide until this step successfully checks out, or you WILL lose access to your server due to the steps later in the guide**):
    * Success: It lists the contents of */root*, which, at the least, should include "*.*" and "*..*". For the purpose of this step, it's okay if it only shows these (even though that's unlikely).
    * Failure: It tells you the following: *user is not in the sudoers file.  This incident will be reported.* In this case, something went wrong with configuring sudo.

From here on out, this guide will assume that you're working as *user* unless otherwise specified. All commands that need *root* privileges or that are going to be run as yet another user (say, *vivo-daemon*) will be run using *sudo*.

## Protecting SSH Access

### Setting up SSH Key Based Authentication

By default, your VPS is most likely configured to allow a simple login by password. Whilst that is convenient and doesn't require any additional setup, it allows everyone to have a shot at trying to log in to your VPS if they manage to guess the password right. Whilst there are ways to mitigate attempts at guessing the password (through a process often referred to as "brute forcing"), making the login process key based lowers the success chance of such attempts into the realm of the stochastically absurd, which is where we want it to be.

#### Linux Desktop

1. Open a terminal.
2. Make sure SSH is installed: *sudo apt-get install openssh-client*
3. Generate the key:* ssh-keygen -t rsa -b 4096*
    1. You will be asked for a location as to where to save the key. Keep in mind that, if you choose a location different from the provided default suggestion, you will have to later specify that key location when using SSH to login to your server. **WARNING: If it prompts you about overwriting**, that means you already have a key. Either use that one, or specify a different location (e.g. */home/user/.ssh/id_rsa_papayajuice*).
    2. Afterwards, it'll prompt you for a passphrase. It's recommended you choose a sentence of sorts that is easy for you to remember, instead of just a short, complicated sequence of symbols. To harden it against directory attacks, you might want to distort the orthographical and grammatical aspects of the sentence a bit, as well as add some special symbols if you're comfortable with that, but make sure it's easy for you to remember nonetheless: If you lose access to this key, it's going to be a hassle.
    3. Make sure to include the entire *.ssh* directory in one of your backups. If your key is stored outside of that, make sure you include that in your backups, too (mind you: in addition, not instead of).
    **Important**: Make sure those backups are as secure as the computer you're using for this, as the passphrase on this key **will** succumb to brute force attacks one day if it falls into the wrong hands. If not in a week, it might in 5 years. Keep that in mind. If you suspect your key got compromised, change it. The passphrase is supposed to give you grace time to do that; it's not an impenetrable bulwark.
4. To configure your server to accept logins with that key, there is a utility called "ssh-copy-id" that can be used for that purpose. The command to do this is as follows (substitute the example IP "1.2.3.4" for your server address):
    * If the default location of the key is unchanged: *ssh-copy-id root@1.2.3.4*
    * If you've specified a different location for the key, the command goes like this: *ssh-copy-id -i /home/user/.ssh/id_rsa_papayajuice user@1.2.3.4*
5. After this, it's time to open a second terminal in order to try logging in to the server with your key, with the usual login commands laid out as follows:
    * Command variations:
        * With the default key location: *ssh user@1.2.3.4*
        * With a user specified key location: *ssh -i /home/user/.ssh/id_rsa_papayajuice user@1.2.3.4*
    * If it asks for your key passphrase instead of your normal login, and logging in with your key passphrase works, that means your server is now configured for key authentication. **WARNING:** If it still asks for your password, something's wrong, and you should **not** proceed with the following steps until it does, else you will lose SSH access to your server.
6. If it works, you're done with this section and now have a working key based SSH login procedure in place.

## Setting up the server side of the Masternode
1. Open a SSH session using your preferred method (e.g. PuttY or a terminal), logging in with *user*.
2. First, we'll be installing some software the VIVO software suite depends on to function, using the following steps:
    1. As VIVO originally forked off of Dash, which traces back to being a fork of Bitcoin, it shares a rather antiquated dependency on an old version of "libdb". To make sure that version is available, we are going to run *sudo add-apt-repository ppa:bitcoin/bitcoin*. This will add a third party software repository containing Bitcoin related packages to the system, such as libdb version 4.8, which is what we need.
    2. We're going to make sure that our software package list is up to date by running *sudo apt-get update*. This is also necessary in order to activate our newly added repository.
    3. To make sure that the system is in an updated state overall before installing anything new, we're going to update it by running *sudo apt-get upgrade*.
    4. Now we'll install the required dependencies in one fell swoop: *apt-get install software-properties-common nano libboost-all-dev libzmq3-dev libminiupnpc-dev libssl-dev libevent-dev libdb4.8-dev libdb4.8++-dev*.
2. Download the newest version of the wallet using these steps:
    1. Go to https://github.com/vivocoin/vivo/releases and copy the link for the **newest** Linux wallet. The copied link should end in *tar.xz*.
    2. Enter the directory you'd like to download to. This guide will assume you're in *user*'s home directory */home/user*. To make sure you are, you can run *cd* (by not specifying a directory to change to, it'll choose the home directory by default).
    3. Download the archive (example link with blind versions): *wget https://github.com/vivocoin/vivo/releases/download/v0.0.0.0/vivo-0.0.0.0-ubuntu14.04.tar.xz*.
    4. Extract the archive, preferably to a location globally accessible (referenced in $PATH, such as */usr/local/bin*), like so: *tar -C /usr/local/bin -xf /home/user/vivo-0.0.0.0-ubuntu14.04.tar.xz*.
    There are a few things to know about this:
        * This will put 4 executables into */usr/local/bin* that can be executed from anywhere by their filename, namely the following:
            * **vivod**:
            The VIVO daemon, which manages the wallet. This will run at all times in order to operate your masternode.
            * **vivo-cli**:
            This is used to interact with the VIVO daemon, such as starting or stopping it, sending coins or looking up the node's status.
            * **vivo-tx**:
            A tool dedicated to transaction management.
            * **vivo-qt**:
            The GUI version of the wallet.
        * If the location you've extracted them to is not globally accessible, you'll have to do either of the following in order to run them:
            * Specify their full path. For example: */home/user/myfavouritedir/vivo-cli*.
            * If you're in the directory you've extracted them into, you can call them using relative path notation, like this (mind the preceding dot, which stands for "current directory"): *./vivo-cli*.
        * This guide's approach to security assumes that the above listed executables are owned by *root* (as a consequence of being extracted by *tar* run by *sudo*, which defaults to root unless it is specified otherwise through the *-u* parameter). Wherever you extract them, be aware of that.
    5. Besides the executable files we've just installed through the above instructions, the VIVO wallet files are all stored in a directory referred to as the "datadir".
        * There are a few things you should know about that:
            * The default name for this directory is *.vivocore*, and its default location is the user's home directory. For the sake of this guide, we'll assume that this translates to the following path: */home/user/.vivocore*.
            * Upon first running the wallet, this directory and various files and other directories inside will be created automatically, which includes "vivo.conf" (full path for the sake of this guide: */home/user/.vivocore/vivo.conf*).
        * We will now run the VIVO daemon once to create these files, and then stop it shortly after to make changes to *vivo.conf* using the following steps:
            1. Run vivod like so: *vivod -daemon*. The *-daemon* parameter will run it in "background mode", which won't block your terminal session or be subject to process termination if your terminal session ends.
                * Note: If you've installed the binaries in a non-global location, that would look like this: */home/user/myfavouritedir/vivod -daemon*.
                * If you intend to use a non-default location for the datadir, follow these steps:
                    1. Make sure the directory exists by creating it first: *mkdir /home/user/mydatadir*.
                        * Unlike in the case of the default datadir not existing, the daemon will complain if an explicitely specified datadir doesn't exist, instead of quietly creating it.
                    2. Start the daemon: *vivod -daemon -datadir=/home/user/mydatadir*.
                        * Make sure to substitute the -datadir parameter with your datadir path every time vivod is called for the node associated with that datadir for every other step in this guide whenever either *vivod* **or** *vivo-cli* are called (e.g. *vivo-cli -datadir="/home/user/mydatadir" getblockcount*).
            2. Wait a few seconds.
            3. Stop the daemon like so: *vivo-cli stop*. Wait until its output confirms daemon shutdown.
                * In case you're using a non-default datadir location, that would look like the following: *vivo-cli -datadir=/home/user/mydatadir stop*.
    6. We will now edit *vivo.conf* with everything needed in order to set up a sound masternode. Start editing it like so: *nano /home/user/.vivocore/vivo.conf*.
        * Observe the following steps:
            1. Copy the following into the (presumably) blank editing field:

                    rpcuser=<Make something up>
                    rpcpassword=<Make something up>
                    rpcallowip=127.0.0.1
                    listen=1
                    server=1
                    daemon=1
                    maxconnections=24
                    masternode=1
                    masternodeprivkey=<Your masternode private key as generated by your local wallet>
                    externalip=<IP of your server>

            2. Replace the values encased in pointy brackets in what you've just copied (don't forget to get rid of the pointy brackets, too). The *externalip* option requires the IP of the server you're working on here and *masternodeprivkey* is the speical masternode key you generate by your local wallet. You can make up whatever you want for the RPC related values. Make it long strings of alphanumeric characters.
            3. Save & close.

    7. Start your masternode by running it again with the appropriate start command you've come to assemble through the above steps. 
