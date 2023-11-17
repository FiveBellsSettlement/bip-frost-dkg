# BIP-DKG (WIP)

This document is a work-in-progress Bitcoin Improvement Proposal for a DKG protocol that can be used in FROST.

## Introduction

### Distributed Key Generation (DKG)

Distributed Key Generation is a protocol between `n` participants that outputs secret key shares for every participant and a shared public key.
Recovering the secret key of the shared public key requires no less than threshold `t` participants to cooperate and use their key shares.
DKG is used as part of the key generation phase in threshold signature schemes like FROST, for example.
In threshold signatures, the shared public key is not recovered.
Instead, the individual key shares are used to sign for the shared public key.
If the DKG succeeds from the point of view of a signer, then FROST signatures are unforgeable, i.e., `t` signers are required to cooperate to produce a signature for the shared public key - regardless of how many other participants in the the DKG were dishonest.

As an alternative of using DKG in threshold signing, a trusted dealer could generate the shared public key and verifiably share the corresponding secret key with the signers.
However, a dishonest or compromised dealer, or not deleting the secret key correctly and getting compromised later, can allow an adversary to forge signatures.

To instantiate DKG there are many possible schemes which differ by the guarantees they provide.
Since DKGs are difficult to implement correctly in practice, the aim of this document is to describe pragmatic DKG protocols that are *simple*, namely SimplPedPop, EncPedPop and RecPedPop.
However, the DKG can be swapped out for a different one provided it is proven to be secure when used in FROST.

Once the DKG concludes successfully, applications should consider creating a FROST signature with all signers for some test message in order to rule out basic errors in the setup.
Moreover, the secret share and shared public key are required by a signer to produce signatures and therefore, signers *must* ensure that they are not lost.
You can refer to the [Backup and Recover](#backup-and-recover) section for additional details.

### Design

- **Large Number of Applications**: This DKG supports a wide range of scenarios. It can handle situations from those where the signing devices are owned and connected by a single individual, to scenarios where multiple owners manage the devices from distinct locations. The DKG can support situations where backup information is required to be written down manually , as well as those with ample backup space. To support this flexiblity, the document proposes several methods to [ensure agreement](#ensuring-agreement), including a potentially novel (?) network-based certification protocol.
- **DKG outputs per-participant public keys**: When DKG used in FROST allowing partial signature verification.
- **Optional instantiation of secure channels for share transfer** (TODO: may not remain optional)
- **Support for backups**
- **No robustness**: Very rudimentary ability to identify misbehaving signers in some situations.
- **Little optimized for communication overhead or number of rounds**
- **Support for Coordinator**: In many scenarios there is a "natural" coordinator who can relay messages between the peers. This reduces communication overhead, because the coordinator is able to aggregate some some messages. A malicious coordinator can force the DKG to fail but cannot negatively affect the security of the DKG.

|                 | seed              | requires secure channels between participants | Requires external Eq | backup                      |
|-----------------|-------------------|-----------------------------------------------|----------------------|-----------------------------|
| **SimplPedPop** | fresh             | yes                                           | yes                  | share per setup             |
| **EncPedPop**   | reuse allowed     | no                                            | yes                  | share per setup             |
| **RecPedPop**   | reuse for backups | no                                            | no                   | seed + transcript per setup |


### Preliminaries

#### Notation

All participants agree on an assignment of indices `0` to `n-1` to participants.

* The function `chan_send(m)` sends message `m` to the coordinator.
* The function `chan_receive()` returns the message received by the coordinator.
* The function `chan_receive_from(i)` returns the message received by participant `i`.
* The function `chan_send_to(i, m)` sends message `m` to participant `i`.
* The function `chan_send_all(m)` sends message `m` to all participants.
* The function `secure_chan_send(i, m)` sends message `m` to participant `i` through a secure (encrypted and authenticated) channel.
* The function `secure_chan_receive(i)` returns the message received by participant `i` through a secure (encrypted and authenticated) channel.
* The function `sum_group(group_elements)` performs the group operation on the given elements and returns the result.
* The function `sum_scalar(scalars)` sums scalars modulo `GROUP_ORDER` and returns the result.
* The function `individual_pk(sk)` is identical to the BIP 327 `IndividualPubkey` function.
* The function `verify_sig(pk, m, sig)` is identical to the BIP 340 `Verify` function.
* The function `sign(sk, m)` is identical to the BIP 340 `Sign` function.

```python
def kdf(seed, ...):
    # TODO
```

#### Verifiable Secret Sharing (VSS)

TODO: the functions `secret_share_shard` and `vss_verify` from the irtf spec are a bit clunky to use for us...

```python
# Copied from draft-irtf-cfrg-frost-15
def polynomial_evaluate(x, coeffs):
   value = Scalar(0)
   for coeff in reverse(coeffs):
     value *= x
     value += coeff
   return value

# Copied from draft-irtf-cfrg-frost-15
def secret_share_shard_irtf(s, coefficients, MAX_PARTICIPANTS):
     # Prepend the secret to the coefficients
     coefficients = [s] + coefficients

     # Evaluate the polynomial for each point x=1,...,n
     secret_key_shares = []
     for x_i in range(1, MAX_PARTICIPANTS + 1):
       y_i = polynomial_evaluate(Scalar(x_i), coefficients)
       secret_key_share_i = (x_i, y_i)
       secret_key_shares.append(secret_key_share_i)
     return secret_key_shares, coefficients

def secret_share_shard(coefficients, MAX_PARTICIPANTS):
    # strip coefficients, strip indices
    shares = secret_share_shard_irtf(s, coefficients, MAX_PARTICIPANTS)[0]
    return [pair[0] for pair in shares]

# Copied from draft-irtf-cfrg-frost-15
def vss_commit(coeffs):
     vss_commitment = []
     for coeff in coeffs:
       A_i = G.ScalarBaseMult(coeff)
       vss_commitment.append(A_i)
     return vss_commitment

def vss_sum_commitments(vss_commitments, t):
    # TODO: using "Lloyd's trick", this optimization should be mentioned somewhere
    n = len(vss_commitments)
    assert(all(len(vss_commitment[0]) == t for vss_commitment in vss_commitments)
    # The returned array consists of 2*n + t - 1 elements
    # [vss_commitments[0][0][0], ..., vss_commitments[n-1][0][0],
    #  sum_group(vss_commitments[i][1]), ..., sum_group(vss_commitments[i][t-1]),
    #  vss_commitments[0][1], ..., vss_commitments[n-1][1]]
    return [vss_commitments[i][0][0] for i in range(n)] +
           [sum_group([vss_commitments[i][0][j] for i in range(n)]) for j in range(1, t)] +
           [vss_commitments[i][1] for i in range(n)]

```

<!-- This is not python -->
```
# Copied from draft-irtf-cfrg-frost-15
def vss_verify_irtf(share_i, vss_commitment)
     (i, sk_i) = share_i
     S_i = G.ScalarBaseMult(sk_i)
     S_i' = G.Identity()
     for j in range(0, MIN_PARTICIPANTS):
       S_i' += G.ScalarMult(vss_commitment[j], pow(i, j))
     return S_i == S_i'
```

```python
def vss_verify(my_idx, vss_commitments_sum, shares_sum):
    return vss_verify_irtf((my_idx, shares_sum), vss_commitments_sum)

# Copied from draft-irtf-cfrg-frost-15
def derive_group_info(MAX_PARTICIPANTS, MIN_PARTICIPANTS, vss_commitment)
  PK = vss_commitment[0]
  participant_public_keys = []
  for i in range(1, MAX_PARTICIPANTS+1):
    PK_i = G.Identity()
    for j in range(0, MIN_PARTICIPANTS):
      PK_i += G.ScalarMult(vss_commitment[j], pow(i, j))
    participant_public_keys.append(PK_i)
  return PK, participant_public_keys
```

### SimplPedPop

We specify the SimplPedPop scheme as described in
[Practical Schnorr Threshold Signatures Without the Algebraic Group Model, section 4](https://eprint.iacr.org/2023/899.pdf)
with the following minor modifications:

- Adding individual's signer public keys to the output of the DKG. This allows partial signature verification.
- Very rudimentary ability to identify misbehaving signers in some situations.
- The proof-of-knowledge in the setup does not commit to the prover's ID. This is slightly simpler because it doesn't require the setup algorithm to take the ID as input.

SimplPedPop requires SECURE point-to-point channels between the participants, i.e., channels that are ENCRYPTED and AUTHENTICATED.
The messages can be relayed through a coordinator who is responsible to pass the messages to the participants as long as the coordinator does not interfere with the secure channels between the participants.

Also, SimplePedPop requires an interactive protocol `Eq` as described in section [Ensuring Agreement](#ensuring-agreement).
While SimplPedPop is able to identify participants who are misbehaving in certain ways, it is easy for a participant to misbehave such that it will not be identified.

In SimplPedPop, the signers designate a coordinator who relays and aggregates messages.
Every participant runs the `simplpedpop` algorithm and the coordinator runs the `simplpedpop_coordinate` algorithm as described below.

```python
def simplpedpop_round1(seed, t, n):
    """
    Start SimplPedPop by generating messages to send to the other participants.

    :param bytes seed: FRESH, UNIFORMLY RANDOM 32-byte string
    :param int t: threshold
    :param int n: number of participants
    :return: a state, a VSS commitment and shares
    """
    coeffs = [kdf(seed, "coeffs", i) for i in range(t)]
    sig = sign(coeffs[0], "")
    # FIXME make sig a separate thing
    my_vss_commitment = (vss_commit(coeffs), sig)
    my_generated_shares = secret_share_shard(coeffs, n):
    state = (t, n)
    return state, my_vss_commitment, my_generated_shares

def simplpedpop_finalize(state, my_idx, vss_commitments_sum, shares_sum, Eq, eta = ()):
    """
    Take the messages received from the participants and finalize the DKG

    :param int my_idx:
    :param List[bytes] vss_commitments_sum: output of running vss_sum_commitments() with vss_commitments from all participants (including this participant) (TODO: not a list of bytes)
    :param scalar shares_sum: summed shares from all participants (including this participant) for this participant mod group order
    :param eta: Optional argument for extra data that goes into `Eq`
    :return: a final share, the shared pubkey, the individual participants' pubkeys
    """
    t, n = state
    assert(len(shares_sum) == n)
    assert(len(vss_commitments_sum) == 2*n + t - 1)
    for i in range(n):
        if not verify_sig(vss_commitments_sum[i], "", vss_commitments_sum[n + t-1 + i]):
            raise BadParticipantError(i, "Participant sent invalid proof-of-knowledge")
    eta += (vss_commitments_sum)
    # Strip the signatures and sum the commitments to the constant coefficients
    vss_commitments_sum_coeffs = [sum_group([vss_commitments_sum[i] for i in range(n)])] + vss_commitments_sum[n:n+t-1]
    if not vss_verify(my_idx, vss_commitments_sum_coeffs, shares_sum):
        return False
    if Eq(eta) != SUCCESS:
        return False
    shared_pubkey, signer_pubkeys = derive_group_info(n, t, vss_commitments_sum_coeffs)
    return shares_sum, shared_pubkey, signer_pubkeys

# TODO: We would actually have to parse the received network messages. This
# should include parsing of the group elementsas well as checking that the
# length of the lists is correct (e.g. vss_commitments are of length t) and
# allow to identify bad participants/coordinator instead of running into
# assertions.
def simplpedpop(seed, t, n, my_idx, Eq):
  state, my_vss_commitment, my_generated_shares = simplpedpop_round1(seed, t, n)
  for i in range(n)
      secure_chan_send(i, my_generated_shares[i])
  chan_send(my_vss_commitment)
  shares = []
  for i in range(n):
      shares += [secure_chan_receive(i)]
  vss_commitments_sum = chan_receive()
  return simplpedpop_finalize(state, my_idx, vss_commitments_sum, sum_scalar(shares), Eq, eta = ()):

def simplpedpop_coordinate(t, n):
    vss_commitments = []
    for i in range(n)
        vss_commitments += [chan_receive_from(i)]
    vss_commitments_sum = vss_sum_commitments(vss_commitments, t)
    chan_send_all(vss_commitments_sum)
```

### EncPedPop

EncPedPop is identical to SimplPedPop except that it does not require secure channels between the participants.
Every EncPedPop participant runs the `encpedpop` algorithm and the coordinator runs the `encpedpop_coordinate` algorithm as described below.

#### Encryption

```python
def ecdh(x, Y, context):
    return Hash(x*Y, context)

def encrypt(share, my_deckey, enckey, context):
    return (share + ecdh(my_deckey, enckey, context)) % GROUP_ORDER
```

#### EncPedPop

The participants start by generating an ephemeral key pair as per [BIP 327's IndividualPubkey](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki#key-generation-of-an-individual-signer) for encrypting the 32-byte key shares.

```python
def encpedpop_round1(seed):
    my_deckey = kdf(seed, "deckey")
    my_enckey = individual_pk(my_deckey)
    state1 = (my_deckey, my_enckey)
    return state1, my_enckey
```

The (public) encryption keys are distributed among the participants.

```python
def encpedpop_round2(seed, state1, t, n, enckeys):
    assert(n == len(enckeys))
    if len(enckeys) != len(set(enckeys)):
        raise DuplicateEnckeysError

    my_deckey, my_enckey = state1
    # Protect against reuse of seed in case we previously exported shares
    # encrypted under wrong enckeys.
    seed_ = Hash(seed, t, enckeys)
    simpl_state, vss_commitment, shares = simplpedpop_round1(seed_, t, n)
    enc_context = [t] + enckeys
    enc_shares = [encrypt(shares[i], my_deckey, enckeys[i], enc_context) for i in range(len(enckeys))
    state2 = (t, my_deckey, my_enckey, enckeys, simpl_state)
    return state2, vss_commitment, enc_shares

def encpedpop_finalize(state2, vss_commitments_sum, enc_shares_sum, Eq, eta = ()):
    t, my_deckey, my_enckey, enckeys, simpl_state = state2
    n = len(enckeys)
    assert(len(vss_commitments_sum) == 2*n + t - 1)

    enc_context = [t] + enckeys
    shares_sum = enc_shares_sum - sum_scalar([ecdh(my_deckey, enckeys[i], enc_context) for i in range(n)]
    # TODO: catch "ValueError: not in list" exception
    try:
        my_idx = enckeys.index(my_enckey)
    except ValueError:
        raise BadCoordinatorError("Coordinator sent list of encryption keys that does not contain our key.")
    eta += (enckeys)
    simplpedpop_finalize(simpl_state, my_idx, vss_commitments_sum, shares_sum, Eq, eta):
```

Note that if the public keys are not distributed correctly or the messages have been tampered with, `Eq(eta)` will fail.

```python
def encpedpop(seed, t, n, Eq):
    state1, my_enckey = encpedpop_round1(seed):
    chan_send(my_enckey)
    enckeys = chan_receive()

    state2, my_vss_commitment, my_generenckeys):
    chan_send((my_vss_commitment, my_generated_enc_shares))
    vss_commitments_sum, enc_shares_sum = chan_receive()

    return encpedpop_finalize(state2, vss_commitments_sum, enc_shares_sum, Eq)

# TODO: explain that it's possible to arrive at the global order of signer indices by sorting enckeys

def encpedpop_coordinate_internal(t, n):
    vss_commitments = []
    enc_shares_sum = (0)*n
    for i in range(n)
        vss_commitment, enc_shares = [chan_receive_from(i)]
        vss_commitments += [vss_commitment]
        enc_shares_sum = [ enc_shares_sum[j] + enc_shares[j] for j in range(n) ]
    vss_commitments_sum = vss_sum_commitments(vss_commitments, t)
    return vss_commitments_sum, enc_shares_sum

def encpedpop_coordinate(t, n):
    vss_commitments_sum, enc_shares_sum = encpedpop_coordinate_internal(t, n)
    for i in range(n)
        chan_send_to(i, (vss_commitments_sum, enc_shares_sum[i]))
```

### RecPedPop

RecPedPop is a wrapper around EncPedPop.
Its advantage is that recovering a signer is securely possible from a single seed and the full transcript of the protocol.
Since the transcript is public, every signer (and the coordinator) can store it to help recover any other signer.

Generate long-term host keys.

```python
def recpedpop_hostpubkey(seed):
    my_hostsigkey = kdf(seed, "hostsigkey")
    my_hostverkey = individual_pk(hostsigkey)
    return (my_hostsigkey, my_hostverkey)
```

The participants send their host pubkey to the other participant and collect received host pubkeys.
They then compute a setup identifier that includes all participants (including yourself TODO: this is maybe obvious but probably good to stress, in particular for backups).

```python
def recpedpop_setup_id(hostverkeys, t, context_string):
    setup_id = Hash(hostverkeys, t, context_string)
    setup = (hostverkeys, t, setup_id)
    return setup, setup_id
```

The participants compare the setup identifier with every other participant out-of-band.
If some other participant presents a different setup identifier, the participant aborts.

```python
def recpedpop_round1(seed, setup):
    hostverkeys, t, setup_id = setup

    # Derive setup-dependent seed
    seed_ = kdf(seed, setup_id)

    enc_state1, my_enckey =  encpedpop_round1(seed_)
    state1 = (hostverkeys, t, setup_id, enc_state1, my_enckey)
    return state1, my_enckey
```

```python
def recpedpop_round2(seed, state1, enckeys):
    hostverkeys, t, setup_id, enc_state1, my_enckey = state1

    enc_state2, vss_commitment, enc_shares = encpedpop_round2(seed_, enc_state1, t, n, enckeys)
    my_idx = enckeys.index(my_enckey)
    state2 = (hostverkeys, setup_id, my_idx, enc_state2)
    return state2, vss_commitment, enc_shares
```

```python
def recpedpop_finalize(seed, my_hostsigkey, state2, vss_commitments_sum, all_enc_shares_sum):
    (hostverkeys, setup_id, my_idx, enc_state2) = state2

    # TODO Not sure if we need to include setup_id as eta here. But it won't hurt.
    # Include the enc_shares in eta to ensure that participants agree on all
    # shares, which in turn ensures that they have the right transcript.
    # TODO This means all parties who hold the "transcript" in the end should
    # participate in Eq?
    eta = (setup_id, all_enc_shares_sum)
    my_enc_shares_sum = all_enc_shares_sum[my_idx]
    return encpedpop_finalize(enc_state2, vss_commitments_sum, my_enc_shares_sum, make_certifying_Eq(my_hostsigkey, hostverkeys), eta)
```

```python
def recpedpop(seed, my_hostsigkey, setup):
    state1, my_enckey = recpedpop_round1(seed, setup)
    chan_send(my_enckey)
    enckeys = chan_receive()

    state2, my_vss_commitment, my_generated_enc_shares =  recpedpop_round2(seed, state1, enckeys)
    chan_send((my_vss_commitment, my_generated_enc_shares))
    vss_commitments, enc_shares_sum = chan_receive()

    shares_sum, shared_pubkey, signer_pubkeys = recpedpop_finalize(seed, my_hostsigkey, state2, vss_commitments_sum, enc_shares_sum)
    transcript = (setup, enckeys, vss_commitments_sum, enc_shares_sum, result["cert"])
    return shares_sum, shared_pubkey, signer_pubkeys, transcript

def recpedpop_coordinate(t, n):
    vss_commitments_sum, enc_shares_sum = encpedpop_coordinate_internal(t, n)
    chan_send_all((vss_commitments_sum, enc_shares_sum))
```

![recpedpop diagram](images/recpedpop-sequence.png)

### Ensuring Agreement
TODO: The term agreement is overloaded (used for formal property of Eq and for informal property of DKG). Maybe rename one to consistency? Check the broadcast literature first

A crucial prerequisite for security is that participants reach agreement over the results of the DKG.
Indeed, disagreement may lead to catastrophic failure.
For example, assume that all but one participant believe that DKG has failed and therefore delete their secret key material,
but one participant believes that the DKG has finished successfully and sends funds to the resulting threshold public key.
Then those funds will be lost irrevocably, because, assuming t > 1, the single remaining secret share is not sufficient to produce a signature.

DKG protocols in the cryptographic literature often abstract away from this problem
by assuming that all participants have access to some kind of ideal "reliable broadcast" mechanism, which guarantees that all participants receive the same protocol messages and thereby ensures agreement.
However, it can be hard or even theoretically impossible to realize a reliable broadcast mechanism depending on the specific scenario, e.g., the guarantees provided by the underlying network, and the minimum number of participants assumed to be honest.

The DKG protocols described above work with a similar but slightly weaker abstraction instead.
They assume that participants have access to an equality check mechanism "Eq", i.e.,
a mechanism that asserts that the input values provided to it by all participants are equal.

Eq has the following abstract interface:
Every participant can invoke Eq(x) with an input value x. When Eq returns for a calling participant, it will return SUCCESS or FAIL to the calling participant.
 - SUCCESS means that it is guaranteed that all honest participants agree on the value x (but it may be the case that not all of them have established this fact yet). This means that the DKG was successful and the resulting aggregate key can be used, and the generated secret keys need to be retained.
 - FAIL means that it is guaranteed that no honest participant will output SUCCESS. In that case, the generated secret keys can safely be deleted.

As long as Eq(x) has not returned for some participant, this participant does not know whether all honest participants agree on the value or whether some honest participants have output SUCCESS or will output SUCCESS.
In that case, the DKG was potentially successful.
Other honest participants may believe that it was successful and may assume that the resulting keys can be used.
As a result, even if Eq appears to be stuck, the caller must not assume (e.g., after some timeout) that Eq has failed, and, in particular, must not delete the DKG state.

More formally, Eq must fulfill the following properties:
 - Integrity: If some honest participant outputs SUCCESS, then for every pair of values x and x' input provided by two honest participants, we have x = x'.
 - Consistency: If some honest participant outputs SUCCESS, no other honest participant outputs FAIL.
 - Conditional Termination: If some honest participant outputs SUCCESS, then all other participants will (eventually) output SUCCESS.
<!-- The latter two properties together are equivalent to Agreement in the paper. -->

Optionally, the following property is desired but not always achievable:
 - (Full) Termination: All honest participants will (eventually) output SUCCESS or FAIL.

#### Examples
TODO: Expand these scenarios. Relate them to SUCCESS, FAIL.

Depending on the application scenario, Eq can be implemented by different protocols, some of which involve out-of-band communication:

##### Participants are in a single room
In a scenario where a single user employs multiple signing devices (e.g., hardware wallets) in the same room to establish a threshold setup, every device can simply display its value x (or a hash of x under a collision-resistant hash function) to the user. The user can manually verify the equality of the values by comparing the values shown on all displays, and confirm their equality by providing explicit confirmation to every device, e.g., by pressing a button on every device.

TODO add failure case, specify entire protocol

Similarly, if signing devices are controlled by different organizations in different geographic locations, agents of these organizations can meet in a single room and compare the values.

These "out-of-band" methods can achieve termination (assuming the involved humans proceed with their tasks eventually).

##### Certifying network-based protocol
TODO The hpk should be the id here... clean this up and write something about setup assumptions

In a network-based scenario, where long-term host keys are available, the equality check can be instantiated by the following protocol:

```python
def make_certifying_Eq(my_hostsigkey, hostverkeys, result):
    def certifying_Eq(x):
        chan_send(("SIG", sign(my_hostsigkey, x)))
        sigs = [None] * len(hostverkeys)
        while(True)
            i, ty, msg = chan_receive()
            if ty == "SIG":
                is_valid = verify_sig(hostverkeys[i], x, msg)
                if sigs[i] is None and is_valid:
                    sigs[i] = msg
                elif not is_valid:
                    # The signer `hpk` is either malicious or an honest signer
                    # whose input is not equal to `x`. This means that there is
                    # some malicious signer or that some messages have been
                    # tampered with on the wire. We must not abort, and we could
                    # still output SUCCESS when receiving a cert later, but we
                    # should indicate to the user (logs?) that something went
                    # wrong.)
                if sigs.count(None) == 0:
                    cert = sigs
                    result["cert"] = cert
                    for i in range(n):
                        chan_send(("CERT", cert))
                    return SUCCESS
            if ty == "CERT":
                sigs = msg
                if len(sigs) == len(hostverkys):
                    is_valid = [verify_sig(hostverkeys[i], x, sigs[i]) \
                                for i in range(hostverkeys)]
                    if all(is_valid)
                        result["cert"] = cert
                        for i in range(n):
                            chan_send(("CERT", cert))
                        return SUCCESS
    return certifying_eq

def certifying_Eq_coordinate():
    while(True):
        for i in range(n):
            ty, msg = chan_receive_from(i)
            chan_send_all((i, ty, msg))
```

In practice, the certificate can also be attached to signing requests instead of sending it to every participant after returning SUCCESS.
It may still be helpful to check with other participants out-of-band that they have all arrived at the SUCCESS state. (TODO explain)

Proof. (TODO for footnote?)
Integrity:
Unless a signature has been forged, if some honest participant with input `x` outputs SUCCESS,
then by construction, all other honest participants have sent a signature on `x` and thus received `x` as input.
Conditional Termination:
If some honest participant with input `x` returns SUCCESS,
then by construction, this participant sends a list `cert` of valid signatures on `x` to every other participant.
Consider any honest participant among these other participants.
Assuming a reliable network, this honest participant eventually receives `cert`,
and by integrity, has received `x` as input.
Thus, this honest participant will accept `cert` and return SUCCESS.

##### Consensus protocol
If the participants run a BFT-style consensus protocol (e.g., as part of a federated protocol), they can use consensus to check whether they agree on `x`.

TODO: Explain more here. This can also achieve termination but consensus is hard (e.g., honest majority, network assumptions...)

### Backup and Recover

Losing the secret share or the shared public key will render the signer incapable of producing signatures.
These values are the output of the DKG and therefore, cannot be derived from a seed - unlike secret keys in BIP 340 or BIP 327.
In many scenarios, it's highly recommended to securely back up the secret share or the shared public key.

If the DKG output is lost, it is possible to ask the other signers to assist in recovering the lost data.
In this case, the signer must be very careful to obtain the correct secret share and shared public key (TODO)!
1. If all other signers are cooperative and their seed is backed up (TODO: do we want to encourage that?), it's possible that the other signers can recreate the signer's lost secret share, .
   If the signer who lost the share also has a seed backup, they can re-run the DKG.
2. If threshold-many signers are cooperative, they can use the "Enrolment Repairable Threshold Scheme" described in [these slides](https://github.com/chelseakomlo/talks/blob/master/2019-combinatorial-schemes/A_Survey_and_Refinement_of_Repairable_Threshold_Schemes.pdf).
   This scheme requires no additional backup or storage space for the signers.

If a signer has the option of deriving a decryption key from some securely backed-up seed and the other signers agree with storing additional data, the signer can use the following alternative backup strategy:
The signer encrypts their secret share to themselves and distributes it to every other signer.
If the signer loses their secret share, it can be restored as long as at least one other signer cooperates and sends the encrypted backup.
