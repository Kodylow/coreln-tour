# A Tour of Core Lightning

Core Lightning (CLN) is a lightweight, highly customizable and standard compliant implementation of the Bitcoin Lightning Network protocol. We'll be running everything in this repl, all the dependencies are set up through Replit's nix backend, but if you're interested in how to install and run Core Lightning on your machine see the walkthrough under "Installation" in [the coreln github repo](https://github.com/ElementsProject/lightning). If you want to run it with Nix like we do here, see [nix-bitcoin](https://github.com/fort-nix/nix-bitcoin), a collection of Nix packages and NixOS modules for easily installing full-featured Bitcoin (and Lightning) nodes with an emphasis on security.

We're going to show you how to:

    1. Set up a CLN regtesting environment to play around in. We'll make 3 lightning nodes with a shared bitcoin backend (more on this later).
    2. Explore the plugin architecture and plug a couple more processes in for additional functionality.
    3. Connect your CLN nodes, fund them in a regtesting environment, and open one way channels.
    4. Perform a collaborative channel open with liquidity ads to dual-fund a channel.
    5. Create and pay bolt11 invoices and bolt12 offers.

Let's get started!

## Setting Up Your CoreLN Regtesting Environment

We've set up the dependencies already for you to run bitcoin and lightning on regtest in this repl. Run the following command to start your bitcoin daemon on regtest for a second (this will build a .bitcoin folder, we need this for the regtest script we're about to run), then shut it down by entering Ctrl-C. 
```
bitcoind -regtest

## You should see something like this:
## 2022-07-03T21:19:15Z Bitcoin Core version v22.0.0 (release build)
## 2022-07-03T21:19:15Z Validating signatures for all blocks.
## 2022-07-03T21:19:15Z Setting nMinimumChainWork=0000...
## ...
```

Niftynei wrote a startup_regtest script (it's also available in the lightning/contrib folder if you're following along on your local machine) that we've copied here that will start up a bitcoin daemon process on regtest with 3 CLN nodes sharing it as a backend. 

Side Note: Lightning nodes don't always need to run their own bitcoin backends, they just need a way to get on-chain data like what the UTXO set is currently and a way to broadcast transactions. There's actually a plugin for CoreLN called "sauron" which lets you start a node that uses blockstream.info's Esplora API to get and send on-chain data. It makes running a lightning node easier but makes you entirely reliant on trusting blockstream!

You can start up the bitcoin and lightning regtesting environment by running the following command in your repl's shell. The script sets a bunch of aliases for your bitcoin and lightning nodes to make typing commands easier:

```
source ./startup_regtest.sh
```

Then spin up your bitcoin backend and 3 lightning nodes by running:
```
start_ln 3
```

If you see the following, you've just spun up your core lightning nodes and your regtest environment is ready to play!
```
Bitcoin Core starting
awaiting bitcoind...
[1] 192 ## this is the process id of cln node1, aliased to l1
[2] 221 ## this is the process id of cln node2, aliased to l2
[3] 255 ## this is the process id of cln node3, aliased to l3
Commands: 
    l1-cli, l1-log,
    l2-cli, l2-log,
    l3-cli, l3-log,
    bt-cli, stop_ln
```

### Exercise A 
Copy down the connection info for your 3 coreln nodes. You can do find them by running `getinfo` for each node like `l1-cli getinfo`, and combining the id, address, and port info as such

```
## id@address:port
## e.g. 038ae5cab4c52ba987dba51536632ae12d90c10138e8e0a0163eae042d03458593@127.0.0.1:7171

## l1 Connection info:

## l2 Connection info:

## l3 Connection info:
```

## So What Exactly IS a Core Lightning Node?

Let's briefly talk about what we mean when we say we 'started our core lightning nodes'. Your Core Lightning "Node" is a node in the lightning network, it is the set of processes which "speak" the lightning protocol. We can add additional processes and functionality to that lightning "node", or remove them sometimes, and this is what brings us to the core lightning architecture of *plugins*.

## Core Lightning Plugins

When you started your core lightning nodes, your terminal printed out the process ids of each of your 3 nodes (they'll be different for you, for me they were 192, 221, 255, see above). Let's look at the process tree, the parent process and child subprocesses, for our first lightning node, by running the following command:
```
pstree 221 ## replace this with your l1's processid
```

Which should output something like...
```
-+= 00221 runner /nix/store/8kgsjv57icc18qhpmj588g9x1w34hi4j-bash-interactive-5.1-p12/bin/bash -rcfile /nix/store/dzpjafx42kx0rmkqyba0m46vpkbjk62r-replit-bashrc/bashrc 
 \-+- 00223 runner lightningd --lightning-dir=/tmp/l2-regtest 
   |--- 00287 runner /nix/store/...clightning-0.10.2/libexec/c-lightning/lightning_gossipd 
   |--- 00284 runner /nix/store/...clightning-0.10.2/libexec/c-lightning/lightning_connectd 
   |--- 00265 runner /nix/store/...clightning-0.10.2/libexec/c-lightning/lightning_hsmd 
   |--- 00247 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/spenderp 
   |--- 00245 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/txprepare 
   |--- 00242 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/pay 
   |--- 00239 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/offers 
   |--- 00238 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/keysend 
   |--- 00235 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/topology 
   |--- 00233 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/funder 
   |--- 00231 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/bcli 
   \--- 00230 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/autoclean
```
(nix dependencies make these kinda hard to read, I've replaced the hash check, pfxk1qxj7p82ziljikwgsxm7h68c0yq2, with '...')

So what's going on here? 

We've got a parent lightningd, aka lightning daemon, process, and almost a dozen child processes, which "plug in" to the parent process and together make up the lightning "node". This is a unique architecture to core lightning, and allows for extreme flexibility and modularity by plugging in and unplugging different processes depending on what we want or need.

Core Lightning ships with a bunch of default plugins, those listed above when you ran pstree, but if we want additional functionality we can plug that functionality into the running lightnigd process.

## Plugging in Our Own Plugins
We've cloned down a bunch of coreln plugins from github to the "plugins" folder in your file tree, feel free to take a look at some of the different plugins. CoreLN itself is written in C, but the modular plugins architecture allows you to write your own plugins in whatever languages you want like Rust, Python, or Go.

We're going to use some of the python ones like "helpme" and "summary" to help us set up and manage our regtest lightning nodes. 

First, we'll install some dependencies for doing lightning network things in python,
```
pip install pyln-client pyln-proto pyln-bolt1 pyln-bolt4 pyln-bolt7
```

Then start the "helpme" plugin for each of our nodes, a fun plugin that helps you set up and use your core lightning node! Notice that we don't have to spin down the nodes, we're plugging in these additional processes with 0 down time.

```
l1-cli plugin start $PWD/plugins/helpme/helpme.py &&
l2-cli plugin start $PWD/plugins/helpme/helpme.py &&
l3-cli plugin start $PWD/plugins/helpme/helpme.py
```

When you start a new plugin, coreln will show you all of the currently running processes including the newly started one. You should see an output that ends with:
```
      ...
         "name": "/nix/store/pfxk1qxj7p82ziljikwgsxm7h68c0yq2-clightning-0.10.2/bin/../libexec/c-lightning/plugins/spenderp",
         "active": true
      },
      {
         "name": "/home/runner/Base58-A-Tour-Of-Core-Lightning/plugins/helpme/helpme.py",
         "active": true
      }
   ]
```
Notice that all of the default plugins are listed, along with our new "helpme" plugin which is now running.

## 'Helpme' Run my CoreLN Node
We're going to run through the helpme instructions for 1 node, l1, but feel free to run through these same steps with your other nodes for additional practice.

Now that we've got helpme plugged in, let's see if it can help us set up and run our lightning node. Run,
```
l1-cli helpme
```

Which should output:
```
Welcome to Core-Lightning!

The lightning network consists of bitcoin channels between computers
(like this one), and the ability to send those bitcoins between them.

It's still beta sofware, so DON'T PUT TOO MUCH MONEY in your lightning
node!  Be prepared to lose your funds (but please report a bug if you do!)

*** You are on TESTNET, not real bitcoin!  See 'helpme mainnet'
STAGE 1 (funds): INCOMPLETE: No bitcoins yet.  Try 'helpme funds'
STAGE 2 (peers): Not connected to the network.  Try 'helpme peers'
STAGE 3 (channels): INCOMPLETE: No channels open.  Try 'helpme channels'
STAGE 4 (making payments): INCOMPLETE: No payments made.  Try 'helpme pay'
STAGE 5 (receiving payments): INCOMPLETE: No payments made.  Try 'helpme invoice'
STAGE 6 (adding bling): You have not customized alias or color.  Try 'helpme bling'
```

So let's walk through it!

### Stage 1: Funding our Lightning Nodes
The first thing we'll need is some bitcoin. Now this is regtest, so we'll have to do things a little differently than on chain. We're in complete control of the regtest environment, so we can mine blocks and generate coins as we like. For simplicity, let's mine coins to the layer 1 bitcoin node, bt-cli, and send from there to our lightning nodes as necessary.

In regtest we use the command `generatetoaddress` to generate a block where the block reward goes to an address we control. Run the following to mine 101 blocks where the coinbase reward goes to the bt-cli's wallet. (we do 101 blocks because coinbase outputs are only spendable after 100 confirmations).
```
bt-cli generatetoaddress 101 $(bt-cli getnewaddress)
```

You should see an array of hashes if this worked, those are the block ids for the blocks you just mined. Check that you've got a 50BTC coin in the wallet by running,
```
bt-cli listunspent ## you'll actually have 2 coins here, the regtest startup script starts at block 1 instead of block 0, but usually when you're running regtests yourself you'll follow this process as we did it here mining 101 blocks
```

So now we've got a bunch of bitcoin on our bt-cli layer 1 node. We have to send that bitcoin to addresses controlled by our lightning nodes. First we generate an address for each of our nodes:

```
l1-cli newaddr && l2-cli newaddr && l3-cli newaddr
```

Then we 

### Stage 2: Connecting to Peers

### Stage 3: Opening Channels

### Stage 3a: Collaborative Channel Opens aka Liquidity Ads

### Stage 4: Making Payments

### Stage 5: Receiving Payments

### Stage 6: Adding Bling???

