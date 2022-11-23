

1. Secure application of sign and encrypt
    - shared data containing the log needs to be encrypted
    - shared data containing the log needs to be authenticated (this allows the recipient to know who sent the data)
    - this requires a secure application of sign and encrypt
    - recommendation of JOSE and Sign&Encrypt: first sign then encrypt
    - however, this introduces the risk of surreptitious forwarding
    - singed data needs to contain the intended recipients

2. Provide metadata for intermediate servers
    - the approach of signing and encrypting ensures confidentiality and authenticity of the shared data containing the log
    - however, from the perspective of intermediate servers, this approach is problematic
    - servers do not learn anything from the ciphertext
    - specifically, they do not know which users might access a log because the intended recipients are only accessible after decryption (the server can not decrypt due to E2EE)
    - this introduces practical problems:
        1.
        - Consider Alice requesting the logs which are intended her
        - The server can not associate logs and users.
        - Thus, the server must reply with all logs stored in the system
        - As a consequence, Alice receives many logs that she can not decrypt
        - But since the raw cipher does not tell her which logs are intended for her, Alice must try to decrypt all logs to find the logs encrypted under her public key.
        - This is very inefficient and requires a lot of computational power.
        2. 
        - Consider Alice to be a data owner.
        - Alice wants to share her log with Bob.
        - Before the sharing process starts, the encrypted log is stored in the server.
        - Alice re-encrypts the log under the public key of Bob.
        - She authenticates against the server an tries to update the stored log in the server.
        - The server, however, does not know if Alice is the legitimate owner of the stored log.
        - Thus, the server can not authorize Alice allowing her to update the stored log.
        - To avoid this, the encrypted log must specify the data owner of the log with the metadata (which are not encrypted).
    - To avoid these problems, the set of receivers and the identity of the data owner needs to be accessible without decryption.
    - Thus, this data can be interpreted as routing information and must be attached as metadata to the encrypted logs.


# Key management in development environment setup

- prepare.sh
    - expects users.json as input
        - sepcifies the users available in dev
    - generates a CA key pair
    - extracts user information and creates crypt keys
        - key pair to sign data
        - key pair to encrypt data
        - public key is signed by a CA private key
    - creates a new users.json which attaches the public keys (e.g. their certificates)

- setup.sh
    - expects a users.json which has attached crypto keys
    - it registers those users in the revolori server
    - relies on the cli of ts-it-crypto to create encrypted logs for users
    - encrypted logs of users are sent to overseer

- container startup
    - development container of clotilde knows all keys
    - once a user logs it it fetches its private keys of this user via the users.json
    - moreover it fetches the ca public certificate which is used to verify if the fetched public keys are signed by a correct CA