flint
======
[中文](README.zh_cn.md)

Simple **experimental** TCP proxy using Enigma rotor cipher applied to base24 encoded data, written in D. The only dependency is [botan](https://github.com/etcimon/botan).

Flint provides strong integrity and **really weak confidentiality**, as Enigma is a WWII cipher. It is recommended to use [stunnel](https://www.stunnel.org/index.html) for some true confidentiality.

Building
------
```
dub build --build=release
```
The example client side config is `flint.config` and server side `flint.config_server`.
You can start the server using `--config=flint.config_server`.

Where are my keys?
------
```
cd keytool
dub --build=release
```
The files pubkey.key and privkey.key will be created under the folder keytool. The server requires privkey.key and the client requires pubkey.key.

Usage
------
Use `--config=<file>` to specify a config file. Explanations go below.

`type` should be `client` or `server`.

`rotors` and `rings` should be the settings of the first, second and third rotors. `reflector` is the type of the reflector. Only 3 rotors are supported currently. See [enigma.d](source/enigma.d) for available types.

On client side, `listen` and `port` specify where to listen for application connections and `remote` and `rport` specify the server address. On server side, `listen` and `port` specify where to listen for clients and `remote` and `rport` specify where to forward applications connections to. `timeout` is the timeout of client or server connections and does not affect application connections. `idletimeout` affects only the server and specifies the length of inactivity before disconnecting a client.

`keyfile` specifies the RSA public or private key file. `powleadingzero` is the required number of leading zero bytes (0x00) in client's proof of work and `powfirstbytemax` is the the highest acceptable value of the first non-zero byte in client's proof of work. `powsalt` is the salt value for proof of work hashes. `maxdisconnectdelay` is the the maximum delay when disconnecting, during which a random delay between 0 and this value will be chosen and the shutdown of connection will only be done after the random delay.

How does it work?
------
Flint multiplexes application TCP connections in one TCP connection. When started, the client does a proof of work and then connects to the server. The first message sent over the connection is the 'hello' message from client to server, which is a 32-byte proof of work string followed by some random alphabetical data. The server checks the proof of work and replies with a 'cookie' message, which is a 8-byte cookie concatenated with a 26-byte alphabet, followed by some random alphabetical data. The client then replies with a 'key' message, which is a base24 encoded RSA cipher string containing crypto keys, mixed with the two letters unused in the base24 process and followed by some random alphabetical data again. After the server's successful decryption, the handshake is finished. The three handshake messages have no length field and flint clearly has broken behavior that an intact message is required to be received at one time. Spaces are always ignored in flint protocol, so an arbitrary amount of spaces could be added into the message being sent over the wire, making flint data stream look more like plain text and enables flint to be a replacement of [bananaphone](https://github.com/david415/bananaphone).

After handshake, the following message structure is used.
```
[HMAC][length authentication tag][length][payload]
```
The message will be encoded using base24 and then encrypted using an Enigma machine. Authenticate then encrypt is a bad idea but I have no idea how to implement encrypt-then-authenticate.
