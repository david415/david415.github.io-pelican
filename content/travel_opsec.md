Title: "data-less" travel opsec
Date: 2014-08-11 16:00
Category: Tails, Tahoe-LAFS
Tags: Tails, Tahoe-LAFS, Tor hidden services, Tor
Authors: David Stainton
Summary: opsec for high risk travel


### Operational security practices for international travel

aka

### Secure Covert Backup Strategy for Tails using Tahoe-LAFS

**note:** You may want to first read [Simple Tails Backup procedure with Tahoe-LAFS](https://david415.github.io/simple-tails-backup-procedure-with-tahoe-lafs.html)
if you are not yet familiar with Tahoe-LAFS



#### context

When traveling in surveilance states where there is risk of search of seizure... **you do not want to be in posession of ciphertext, key material or other sensitive data**. If you are detained and have ciphertext they can request you for the phassphrase to decrypt your data. Things may not go so well for you if you don't comply with their demands.

To fulfill this objective we can reduce the complexity of 
restoring our confidential backup to two pieces of information
that can be remembered (and should not be written down):

1. passphrase to unlock critical blob
2. retreival information of critical blob

Once you've setup several redundant hiding places for your critical ciphertext blob of Tahoe-LAFS information then you
"secure erase" your drive(s) before traveling.


#### Create your critical ciphertext blob like this:

This critical blob contains Tahoe-LAFS cryptographic capabilities and grid connection information.
That is... everything we need to restore all our sensitive data; key material and configuration files in my case.
It's a total loss if an attacker gains access to our Tahoe-LAFS cryptographic capabilities.
We therefore protect this data by symmetrically encrypting with an entropic passphrase.

Create a textfile containing Tahoe-LAFS onion grid connection information and root cryptographic capability:

```bash
cat <<EOT>critical
export introducer_furl='pb://MyTubID@MyOnionAddress.onion:OnionPort/SuperSecretSwissnum'
root_cap='URI:DIR2:aaaaaaaaabcaaaaaaaaaaaaaaa:kkkkkkkkkkkkkkfghkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkk'
[client-server-selection]
server.v0-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.type = tahoe-foolscap
server.v0-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.nickname = RobotKillHuman
server.v0-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.seed = aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
server.v0-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.furl = pb://aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@cccccccccccccccc.onion:34273/cswisssssssssssssssssssssssssnum
server.v0-bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb.type = tahoe-foolscap
server.v0-bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb.nickname = RobotPlanetConspiracy
server.v0-bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb.seed = bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
server.v0-bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb.furl = pb://bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb@dddddddddddddddd.onion:34273/bswisssssssssssssssssssssssssnum
server.v0-cccccccccccccccccccccccccccccccc.type = tahoe-foolscap
server.v0-cccccccccccccccccccccccccccccccc.nickname = StorageForRobotsOnly
server.v0-cccccccccccccccccccccccccccccccc.seed = cccccccccccccccccccccccccccccccc
server.v0-cccccccccccccccccccccccccccccccc.furl = pb://ccccccccccccccccccccccccccccccc@eeeeeeeeeeeeeeee.onion:34273/aswissssssssssssssssssssssssssnum
EOT
```

Unfortunately we have not yet merged Leif Ryge's [truckee feature branch](https://github.com/leif/tahoe-lafs/tree/truckee) of Tahoe-LAFS upstream.
Use this truckee so that you have my **introducer-less** feature which allows you to specify storage node connection information
directly in your **tahoe.cfg**... thus eliminating the introducer node as the single point of failure and success.

Currently I use DJB's NaCl SecretBox to verifiably encrypt/decrypt the critical connection
information and root cap like this:
[secretBox.py](https://github.com/david415/hidden-tahoe-backup/blob/master/HiddenTahoeBackup/secretBox.py)
```bash
./secretBox.py --encrypt critical > critical.secretbox
```

Make sure to pick a long entropic passphrase with enough interesting words.


#### What does retrieval information of critical blob mean?

Your threat model may permit you to publicly stash your critical blob
in a memorable location such as:

* on a gist or github repo
* in a tweet
* in a blog post
* web host at memorable url

On the other hand your threat model may advise against exposing your critical blob to cryptanalysis...
You might prefer more covert methods of recovery:

* use ssh to retreive the file from a private shell account
* Pond message a friend; retreive from friend directly
* PGP e-mail a friend; retreive from friend directly



### Restore procedure:

1. acquire Tails disk
2. acquire critical ciphertext blob
3. decrypt ciphertext blob
4. configure Tahoe-LAFS client
5. download all your private data

