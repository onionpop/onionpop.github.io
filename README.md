## Privacy-Preserving Onion Site Popularity Measurement

This page provides details about a small, short-term, safe, and secure measurement study of the popularity of [Tor](https://www.torproject.org) onion sites. In particular, we aim to show that it is feasible to estimate the popularity of onion sites by using machine learning techniques to classify accesses to the Facebook onion site front page, while providing strong user privacy guarantees.

We believe that this study is safe: we only learn the total site usage, with added noise. Because we collect data from Tor middle relays, we can not learn the identity of individual users. Because we use PrivCount, we do not learn site usage on single relays, or total site usage without any noise. All unblinded data is kept on the collecting relay, and is automatically destroyed after processing. See the [safety section](#what-are-we-doing-how-is-this-safe) for more details.

### Who are we?

The results of this work will appear in the following publication:

```
Applying Traffic Fingerprinting to Measure Tor
25th Symposium on Network and Distributed System Security (NDSS 2018)
Rob Jansen, Marc Juarez, Rafael Galvez, Tariq Elahi, and Claudia Diaz
```

### What is the general idea?

We are working on understanding how machine learning techniques that have traditionally been used to fingerprint websites for the purpose of deanonymizing Tor users can instead be used to measure the popularity of Tor onion sites. Although website fingerprinting is usually done by someone in a position to observe part of the network path between the client and its guard (including someone that runs the guard itself), our popularity measurement is less intrusive and only requires running middle relays.

Our popularity measurement involves:
  1. running a middle relay and observing circuits;
  1. guessing if an observed circuit is a hidden service circuit (already done by [Kwon et al.](https://www.usenix.org/node/190967), albeit from a guard node position);
  1. for hidden service circuits, guessing the position of the relay in the circuit (we focus on the middle position next to the
client-side guard); and
  1. for hidden service circuits observed from the correct relay position, guessing the onion service website (i.e. the Facebook login page) based on a trained classifier.

If guessing the onion site from the middle is successful, then it can be used to discover onion service popularity by measuring the frequency with which each onion site is accessed. The goal of our small study is to provide a proof-of-concept by measuring the popularity of a single onion page: the Facebook onion site login page.

### What are we doing? How is this safe?

Our measurement study explicitly **prioritizes user safety as a primary goal**. We practice data minimization, limit measurement granularity, and provide additional security to the measurement process as described below. We have incorporated feedback from the [Tor Research Safety Board](https://research.torproject.org/safetyboard.html) into our methodology.

Because this measurement is done from the middle relay position, **onion-encryption technically prevents us from learning any client-identifying information**. Although this protects users to some extent, we further protect users by utilizing the state-of-the-art in safe Tor measurement tools and techniques. Specifically, we use [PrivCount](https://github.com/privcount) and the techniques set out by [Jansen and Johnson in "Safely Measuring Tor"](http://www.robgjansen.com/publications/privcount-ccs2016.pdf) to provide differential privacy and securely aggregate measurements across all of our relay data collectors.

During the measurement process, circuit and cell metadata will be used by the classifiers to make their guesses. Circuit meta-data includes internal middle relay state that is used to identify the previous and next relay in the circuit (the circuit ID, channel ID, and public relay fingerprint and flags). Cell metadata includes whether the cell was sent or received and to/from which side of the circuit, the previous and next circuit ID and channel ID, the cell type and relay command type (if known), and a timestamp of when the cell was sent or received.

The meta-data is sent in real time from Tor to PrivCount where it will be temporarily stored in volatile memory (RAM); the longest time that PrivCount will store the data in RAM is the lifetime of the circuit (normally around 10 minutes). When the circuit closes, PrivCount will pass the meta-data to a previously-trained classifier, which will make the guesses as appropriate. PrivCount will increment basic circuit counters that will allow us to compute the following statistics:

  1. The fraction of all middle circuits that we predict are client-side rendezvous circuits
  1. The fraction of above, in which we predict that our middle is the second relay from the client (next to the client's guard)
  1. The fraction of above, in which we predict the circuit was used to access the Facebook onion site front page

During our measurement, we visit the Facebook onion site front page with our own client to generate some ground truth circuits which we can use to check the accuracy of our predictions. Additionally, we collect some counters when our relays serve in the rendezvous position that can also be used to check our predictions. These include:

  1. The fraction of entry/middle/exit circuits that are rendezvous circuits
  1. The fraction of rendezvous circuits that connect to a known Facebook ASN

Once these counters are incremented, **all meta-data corresponding to the circuit and its cells are destroyed**.

The PrivCount counters are initiated to noisy values to ensure differential privacy is maintained (cf. ["Safely Measuring Tor"](http://www.robgjansen.com/publications/privcount-ccs2016.pdf)), and are then blinded and distributed across several share keepers to provide secure aggregation. At the end of the process, **we learn only the value of these noisy counts aggregated across all data collectors, and nothing else about the information that was used during the measurement process**. Specifically, we do not learn relay-specific inputs to the final counter value, and client usage of Tor during our measurement will be protected under differential privacy (i.e., the final counter values include noise to protect the true counts).

### Why are we doing this?

This work has value to the community that we believe offsets the potential risks associated with the measurement.

  - We highlight the positive use of Tor and onion services by focusing our measurement on the Facebook onion site. Understanding which parts of the Tor protocol are used most often can help Tor researchers and developers focus their effort on improvements that can have the largest impact on the widest set of users.
  - We believe that showing how website fingerprinting can be applied for purposes other than client deanonymization (the focus of most recent website fingerprinting research) is novel and interesting and may spur additional research that may ultimately help us better understand the real world risks associated with fingerprinting techniques. This may, in turn, lead to the development of better fingerprinting defenses.
  - We believe that the risk from middle nodes is too often overlooked in the literature, and we think there is value in showing what can be discovered from the relay position with the fewest requirements.

### Which relays are involved in the measurement?

The following relays are part of our PrivCount deployment. The data collected by these relays is blinded and securely aggregated while also being protected by differential privacy. Each individual relay learns nothing except the final combined results from all relays.

  - [09FA8B4F665AD65D2C2A49870F1AA3BA8811E449](https://atlas.torproject.org/#details/09FA8B4F665AD65D2C2A49870F1AA3BA8811E449)
  - [068308AD070849A71B8C1DB06C2509E82C40B908](https://atlas.torproject.org/#details/068308AD070849A71B8C1DB06C2509E82C40B908)
  - [335746A6DEB684FABDF3FC5835C3898F05C5A5A8](https://atlas.torproject.org/#details/335746A6DEB684FABDF3FC5835C3898F05C5A5A8)
  - [363F42695F2DD825DA5A4E6ABF3FBDFCFD1E9AE2](https://atlas.torproject.org/#details/363F42695F2DD825DA5A4E6ABF3FBDFCFD1E9AE2)
  - [B6718125C43ECA2E5011B3C681BB6638617A9686](https://atlas.torproject.org/#details/B6718125C43ECA2E5011B3C681BB6638617A9686)
  - [C6B3546CC6BCCB649FEC82D348D464554BC6323D](https://atlas.torproject.org/#details/C6B3546CC6BCCB649FEC82D348D464554BC6323D)
  - [12B80ABF019354A9D25EE8BE85EB3C0AD8F7DFC1](https://atlas.torproject.org/#details/12B80ABF019354A9D25EE8BE85EB3C0AD8F7DFC1)
  - [DE684E6C6B7773B8BE74B4D941E4178988E15E26](https://atlas.torproject.org/#details/DE684E6C6B7773B8BE74B4D941E4178988E15E26)
  - [4B1E3276137AD12DCCEBE354EA11C1E47F804F67](https://atlas.torproject.org/#details/4B1E3276137AD12DCCEBE354EA11C1E47F804F67)
  - [C170AE5A886C5A09D6D1CF5CF284653632EEF25D](https://atlas.torproject.org/#details/C170AE5A886C5A09D6D1CF5CF284653632EEF25D)
  - [A5945077E0D35729F8E2920A54BE12A0058B403E](https://atlas.torproject.org/#details/A5945077E0D35729F8E2920A54BE12A0058B403E)
  - [0DA9BD201766EDB19F57F49F1A013A8A5432C008](https://atlas.torproject.org/#details/0DA9BD201766EDB19F57F49F1A013A8A5432C008)
  - [D53793315E290D250E9AFC431A4C9068A1E53C98](https://atlas.torproject.org/#details/D53793315E290D250E9AFC431A4C9068A1E53C98)
  - [11EAB5C9137906EF7E6A32365C4B37613698E647](https://atlas.torproject.org/#details/11EAB5C9137906EF7E6A32365C4B37613698E647)
  - [91516595837183D9ECD1318D00723A8676F4731C](https://atlas.torproject.org/#details/91516595837183D9ECD1318D00723A8676F4731C)
  - [1A4488A367D89D0EFDA88116059FEBCACF0F508A](https://atlas.torproject.org/#details/1A4488A367D89D0EFDA88116059FEBCACF0F508A)
  - [98D10461F6EDF13780D20D7E402E67F40C5ADBD9](https://atlas.torproject.org/#details/98D10461F6EDF13780D20D7E402E67F40C5ADBD9)

### What is the measurement status?

Current status:
```diff
- We are not currently running a measurement phase
```

Previous status:
```diff
+ Measurement 1: start time `2017-08-02 01:54:12 UTC`, end time `2017-08-03 01:54:12 UTC`
+ Measurement 2: start time `2017-08-07 01:23:43 UTC`, end time `2017-08-08 01:23:43 UTC`
+ Measurement 3: start time `2017-08-09 15:01:59 UTC`, end time `2017-08-10 15:01:59 UTC`
```
