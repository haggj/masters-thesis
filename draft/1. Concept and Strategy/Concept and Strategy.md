# Concept and Structure

1. [Init System](#init-system)
2. [Elaborate Changes in Toolchain](##elaborate-changes-in-toolchain)


0. Abstract
1. Introduction
2. Requirements
3. Survey of potential solutions
4. Approach
5. Evaluation
6. Conclusion

# 1. Init system

## 1.1. Checkout toolchain and start servers locally
Checking out and building:
- checkout ssh not possible
- Revolori container does not build because `go get` does not install anyore (since GO version 1.18). Needs to be replaced with `go install`
- issues because docker on M1 does not support ```network_mode: host```
- using ```multipass``` to run docker-compose in virtual ubuntu environment
- to access containers from Mac (see details in `toolchain/run/dev/multipass_forward` script):
  Important command:       
  ```bash
  socat TCP-LISTEN:5421,fork,bind=$from TCP:$to:5421 &
  ```
   - port forwarding from 127.0.0.1 to 192.168.64.5 (address of    multipass guest OS) within Mac host OS
   - port forwarding from 192.168.64.5 to 127.0.0.1 within multipass guest OS
   - Disable HostOrigin checks in Revolori via 
      ```go
      AllowOriginRequestFunc: func(r *http.Request, origin string) bool { return true },
      ```
   - Configured Pycharm to use remote SSH interpreter. How this works:
       - Within multipass a venv is created (via ```pipenv install```) 
       - The venv created is located on the ubuntu machine (can be seen if starting the shell via ```pipenv shell```)
       - The Pycharm remote interpreter points to this venv within the ubuntu guest


## 1.2. Initialize Latex template


# 2. Elaborate changes in toolchain

## 2.1. Current data flow as diagram
Description of components:
![](images/OverviewComponents.jpg)
![](images/Auth.jpg)
- Revolori
  - Singe-Sign-On Server
  - authenticating users which access data and users which request their data logs
  - based on ordinary user certificates
- Coltide
  - "Disyplay"
  - making the usage information transparent to the user
- Oversser
  - "Safekeeper"
  - verifying and stroing usage information
- Valut

Note that there is no explicit Monitor component. Monitors could be for example an issue tracking software (e.g. Jira). Therefore a lightweight library needs to be added to these tools to ensure data accesses are tracked correctly
Company has control over the servers where the toolchain is deployed -> "needs to be trusted in some way"  
**important question regarding section 4.2.2: Does the admin have access to the root keys (e.g. can the admin decrypt, encrypt, modify everything)?**
  
## 2.2. Define goals

### 2.2.1 What are the goals of this thesis?

There are two main goals in the context of this thesis:
1. Encrypt log data, s.t. overseer can not access logs (confidentiality)  
   Defending against `hontest-but-curions server`:
      - attacker follows protocol of overseer to store and load data
      - attacker attempts to analyze all legitimately received information (e.g. analyzes logs and policies stored in the database)
      - can collude with others (e.g. a malicious data owner)
      - passive attacker
      - similar to section 12.2 in https://sci-hub.ru/https://doi.org/10.1016/B978-0-12-805394-2.00012-X


2. Data owner needs to be able to share and revoke log data with others while defending against `malicious data owner`:
      - attacker does not follow the protocol and tries to manipulate data logs (integrity and authenticity)
      - attacker tries to share manipulated logs with others (e.g. because attacker wants to discredit others)
      - active attacker
      - similar to section 3.1 https://mediatum.ub.tum.de/doc/1615228/7emnvw4k3nyma9pbthb37cnc4.ease2021-29.pdf

**Functional requirements**
 1. Once a log was created, only the owner can access the log.
 2. The owner of a log can share the logs with other users within the system.
 3. The owner of a log can revoke the access to the logs.

**Non-functional requirements**
1. Security 
    - simplicity
    - standardized crypto
    - reduce number of trusted entities
2. Usability
    - UI handles signature checks
    - avoid complex user actions
    - visualization of shared data
    - avoid outdated data
3. Performance
    - minimal resource utilization (avoid redundant data)

**Easy solution:**  
Each entity has a public-key pair.  Each data log is encrypted with the public key, s.t. only data owner can decrypt (resolves goal 1). Goal 2 can be solved by encrypting signed data. After each decryption this signed data need to be verified.

**Problems:**
1. How could data be shared, e.g. how to resolve goal 2 if data logs can only be decrypted by the data owner?
2. How can encrypted data be stored/queried on the server?


**Ideas regarding problem 1**:
- **External Encryption:** Encrypt data log with public key of target and send externally (e.g. via email)

  - #️⃣ Security
    - ❌ encryption is responsibility of user -> error prone 
    - ✅ can be realized using standard crypto 
    - ✅ simplicity
  - ❌ Usability: 
    - ❌ not user-friendly, because decryption, signature checks and visualization needs to be handled externally
    - ❌ encryption is responsibility of user -> error prone
  - ❌ Data quality: 
    - ❌ data changes not reflected in already shared data
    - ❌ redundant data in the system if logs are shared with multiple entities

- **Mutual Encryption:** Data owner re-encrypts the log for the target. The re-encrypted data is sent back to overseer, which saves them in a "sharedLog" table

  - ✅ Security
    - ✅ can be realized using standard crypto 
    - ✅ simplicity
  - ✅ Usability: 
    - ✅ data is handled within the system 
  - ❌ Data quality: 
    - ❌ data changes not reflected in already shared data
    - ❌ redundant data in the system if logs are shared with multiple entities


  **Challenges**:  
  How to access private keys of users? 

- **Key Server:** Encrypt logs with unique symmetric key and store them in database. The symmetric key is stored on a trusted key server, which hands out the key to authorized users: After creation only the owner is allowed to access the key. The owner can add/revoke permissions of others to access the key.

<img src="images/key-server.png" width="500"/>

  - #️⃣ Security
    - ✅  can be realized using standard crypto
    - #️⃣ requires a key server
    - #️⃣ getting more complex because during each decryption key server needs to be requested
  - ✅ Usability: 
    - ✅ data is handled within the system 
  - ✅ Data quality: 
    - ✅ each log is only stored once


- **Broadcast Encryption** Re-encrypt data by means of a broadcast encryption scheme. Only a defined set of users can decrypt the data.
<img src="images/broadcast-encryption.png" width="500"/>

  - #️⃣ Security
    - #️⃣ requires a key distribution center (e.g. a trusted entity, could be included into revolori)
    - #️⃣ not standard crypto: requires porting of crypto libraries (at least to JS)
  - ✅ Usability: 
    - ✅ data is handled within the system 
  - ✅ Data quality: 
    - ✅ each log is only stored once

  
  **Challenges**:  
  - How to access private keys of users?
  - How to implement ABE in frontend?
    - critical part: implement bilinear pairing
      - C: https://crypto.stanford.edu/pbc/
      - Python: https://pypi.org/project/charm-crypto/ (using C lib under the hood -> not possible to use this via brython)
      Curve definitions: https://github.com/JHUISI/charm/blob/acb55513b244bfdebbfe715cec7b564c8e850779/charm/toolbox/pairingcurves.py
      - Rust: https://github.com/zcash-hackworks/bn
      - Rust (Fraunhofer): https://github.com/georgbramm/rabe-bn 
    - use web assembly to implement bilinear pairing efficiently in browser:
      1. Follow the reference implementation (e.g. AC17 in python, which relies on pbc)
      2. Outsource heavy computation to wasm (pairing) or use existing library (need to be converted to WASM)
       
**Ideas regarding problem 2**
- if solving problem 1 with a dedicated crypto scheme, chances are high that this crypto scheme does not support queries over ciphertexts
- schemes that allow queries over encrypted data are currently researched (e.g. homomorphic encryption)
- Alternative: Keep a relational database
  - encrypt only logs (e.g. as a json)
  - save id of owner along with encrypted logs
  - save ids of shared owners with encrypted logs (1:n realtion)
  - improvement: provide pseudonyms for user ids, which are managed by revolori
      
### 2.2.2. What needs to be implemented?
#### Visualize desired data flow as diagram
#### Which components are involved?
#### Are additional components necessary?
#### Which technology can be used?
#### Reasons and alternatives?
### 2.2.3. Current vs desired data flow
#### which existing components need to be adjusted?
#### which new components need to be created?

