---
date: '2026-11-07T06:56:00-05:00'
draft: false
title: "For my muggle friends: Taking your bitcoin off the exchange"
---

I want bitcoin to be for everyone. Still, anyone who owns it, whether they hold it themselves or leave it on an exchange, should understand the risks and responsibilities that come with it.

In July, Coinkite disclosed that a [firmware bug in some Coldcard hardware wallets had weakened the randomness behind the seeds they generated](https://blog.coinkite.com/coldcard-mk3-seed-generation-warning/), and attackers recreated private keys from those seeds. In October, Ledger reported that [criminals had opened genuine Ledger devices, added hidden components that captured recovery phrases during setup, and put them back on sale](https://www.ledger.com/blog-response-hardware-implant-attack). Last week, [Binance paused withdrawals after attackers drained its hot wallets](https://www.cnbc.com/2026/10/30/billions-in-crypto-frozen-in-binance-hack-heres-what-happens-next-.html).

If you bought bitcoin on an exchange, the exchange holds it for you. The balance in your app is the company's promise to pay, and the company holds the keys, which are the secrets that authorize spending.

Once your bitcoin sits in a wallet whose keys you control, no exchange can freeze it or lose it in a hack, but nobody can reset your password either. If you lose both the keys and your backup, the bitcoin is gone, and if someone else gets either one, they can spend it.

So I treat the backup as the most important part. Before anyone moves anything meaningful, I want them to have restored from it at least once, and to keep it somewhere safe from fire, hard to lose, and away from anyone who shouldn't see it.

What about hardware wallets? They're devices built to do one thing, which is why bitcoiners love them. A hardware wallet removes your dependence on the exchange too, but it adds other risks such as the maker's supply chain, its firmware, and, as the Coldcard bug showed, the randomness it uses to create your keys. Every setup carries some risk, so I'd ask you which risks you're willing to take and whether you can protect against them. In that sense, bitcoin is a choose-your-own-adventure.

The path in this post uses a general-purpose device running a general-purpose operating system: a smartphone. Some people will tell you that's reckless. Phones do have known weaknesses, but the precautions below (a factory reset, keeping the keys on a phone that stays offline, and a backup you've practiced restoring from) reduce those risks a great deal.

A few of my friends have asked me what I would do, so I've written down the path I'd guide them through. You can do this without becoming a developer if you follow a good guide and test things before trusting them.

## BlueWallet on a spare phone

You'll need your everyday phone and a spare iPhone or Android phone, like an old one in a drawer. BlueWallet's guide to [using BlueWallet offline as a cold wallet](https://bluewallet.io/docs/offline-cold-wallet/) shows you how to turn the spare phone into a signer. Signing is how your keys approve a payment, and a signer is a device whose only job is to do that. The spare phone holds the private keys and stays offline. Your everyday phone runs a [watch-only wallet](https://bluewallet.io/watch-only/) that holds only the public keys, which is enough to show your balance, generate addresses, and prepare transactions for the spare phone to sign.

BlueWallet's App Store listing asks for iOS 15.6 or later, and on Android, Google Play shows whether the phone is compatible. Both phones need a working camera, since the simplest way to pass the public key and each transaction between them is a QR code.

The cold wallet guide has you factory reset the spare phone, take out the SIM card, and install BlueWallet while the phone is still online. Then you switch on Airplane Mode and BlueWallet's Offline Mode (Settings > Network > Electrum Server), create the wallet, and write down your recovery phrase (a list of words that can rebuild the wallet). Under Wallet details, Show Wallet XPUB puts the wallet's public key on screen as a QR code, and you scan it into BlueWallet on your everyday phone to create the watch-only wallet. The companion page, [Sign a transaction offline](https://bluewallet.io/docs/sign-offline/), covers each payment after that: the watch-only wallet builds the transaction and shows it as a QR code, the spare phone scans and signs it, and the everyday phone scans the signed transaction back and broadcasts it to the network.

I'd add a few steps of my own. I'd use a phone that has only ever been yours, since a reset won't fix a phone you already have reason to distrust. I'd copy off anything worth keeping before the reset, install the latest system update the phone will accept before BlueWallet goes on, and check that Wi-Fi, mobile data, and Bluetooth are all off once Airplane Mode is on. The guide is blunt about one rule I'd repeat: once the keys are on that phone, never turn off Airplane Mode or Offline Mode, not even "just for a moment."

The recovery phrase is your backup. I'd write it by hand on paper and never photograph it.

## Moving your bitcoin off the exchange

Blockstream's short article, [How and why should I withdraw my bitcoin from an exchange?](https://help.blockstream.com/education/wallets/managing-and-acquiring-bitcoin/how-and-why-should-i-withdraw-my-bitcoin-from-an-exchange), covers the basics, including why a withdrawal can take a while (exchanges often batch them and may run compliance checks). I'd do it in this order:

1. Test your backup.
2. Open the watch-only wallet on your everyday phone and tap Receive to get an address. BlueWallet's [receiving guide](https://bluewallet.io/docs/receive-bitcoin-transaction/) walks through it.
3. In the exchange's withdrawal form, choose Bitcoin on-chain if it asks for a network, paste the address, and compare it with the one in your wallet character by character, all the way to the end.
4. Withdraw a small amount and wait for your wallet to show it.
5. Send part of it to a new address in your own wallet to prove you can spend it.
6. Move the rest, in one go or in stages.

## The next step: 2-of-3 multisig

Once a single-key wallet feels routine, I'd strongly encourage my friends to take the next leg of the adventure, which is 2-of-3 multisig. You keep three keys in separate places, and any two can authorize a payment, so one lost or stolen key no longer costs you the bitcoin. But you also have to back up the wallet setup that ties the keys together, and as you can imagine it gets more complex. If you want to read ahead, BlueWallet's [Multisig Vault page](https://bluewallet.io/multisig-wallet/) covers its 2-of-3 Vaults: keeping the keys on separate devices, the two backups you'd store, and a demo of two phones signing offline.

In the coming days, I'll write a follow-up post with a few beginner-friendly 2-of-3 setups, some through companies that make it easy and some you set up yourself. I'll cover the risks of each and where it gives up some "sovereign safety" for convenience.
