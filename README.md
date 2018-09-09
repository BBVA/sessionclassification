# Introduction
## Theory

In cybersecurity, one of the branches of analytics that is gathering strength is to analyze the behavior of users, machines, services, etc. One way would be to model the behavior of ports depending on whether traffic passing through it is encrypted or not. If a time series analysis process were performed, whose value is the percentage of encrypted traffic passing through each port, anomalies could be detected in various phases of a Cyber Kill Chain.

In this repository the first step is taken, an algorithm is offered that determines if a session is encrypted. To fulfill this purpose, the network protocol it uses can be used as a base, which is not reliable, since encryption layers can be added regardless of the protocol used. It would also be assumed that the tools that are available understand perfectly all the protocols and can read their headers, even if they are protocols that are not public like the one used by Citrix. Also keep in mind that as new versions of the protocols come out, the tools should be updated and understood.

In addition, encryption can be confused at a glance with serializations, with encodings, with protocol headers that can not be interpreted by PCAPS parsing tools (Wireshark, Scapy, etc.).

A very extended tool to determine if a file or a string is encrypted is <a href="http://www.fourmilab.ch/random/"> Ent </a>, which provides the following metrics:

- Entropy
- Chi-square Test
- Arithmetic Mean
- Monte Carlo Value for Pi
- Serial Correlation Coefficient


A good technique to determine if a file is encrypted, is to calculate a probability distribution with the bytes it contains. In the paper <a href="https://www.researchgate.net/publication/50392274_Substitution-diffusion_based_Image_Cipher?_sg=tGAmmy34BimyDZ2PgSk-pPO_aZxQG7cUFF_sRmSelPmf0gYLs7ocBt4Ew0NyuSgRyu3VZFMbhg"> of Narendra  K Pareek, Vinod Patidar and Krishan K Sud called *'Substitution-diffusion based Image Cipher'* </a> shows how the probability distribution of the bytes of an image changes when it is encrypted.
  

<div align="center" >
    <img width="70%"  src="img/hist-cifrado.png"/>
</div>

It is observed that the distribution is flattened, becoming much more uniform than when they are encrypted, which implies that each byte would be equiprobable and the average would be centered. This makes the metrics proposed by Ent very effective to know if a file or a string is encrypted, since many of them are based on knowing the uniformity of the probability distribution of the bytes that make up the file in question.



## Practise


The theory says that an encrypted content should follow a uniform distribution. However, in the case of network packets, it is not so simple to observe for various reasons:
- Headers of the application layer protocols not properly parseed skew the distribution, putting, in most cases, many bytes to 0.
- Content encrypted but very small, so only one or several peaks are observed.
- A TCP session always starts with a handshake that never goes encrypted, but then they can exchange encrypted content. Giving a result per session will not result in a uniform distribution.

Below is an example of how the SMB3 protocol shows some of the mentioned problems. This protocol contains data in your payload encrypted with AES-128-CCM.

There are protocols of the application layer whose headers do not understand either of them. As seen in the image below, the payload includes the red area, which corresponds to the NetBios layer, when the relevant area is green. It should be noted that most of the bytes that are included are unduly zeros.
It is the deepest payload (that of the application layer) to which both Scapy and Wireshark do not allow access in many cases.

<div align="center" >
  <img width="70%"  src="img/smb3-aes-128.png"/>
</div>

In the case of the PCAP with packages whose application layer uses the SMB3 protocol, which contains encrypted data, the probability distribution can be visualized, which theoretically should be uniform, since the data is encrypted. However, headers produce a distortion in the probability distributions, as seen in the following.

<div align="center" >
  <img width="70%"  src="img/distribucion-smb-aes-128.PNG"/>
</div>
 
One way to prevent these distortions from affecting the algorithm's effectiveness is to eliminate certain values from the distribution. The result is as follows:

<div align="center" >
  <img width="70%"  src="img/smb3-aes-128-cut.png"/>
</div>


In this way, it is achieved that the algorithm can identify whether a TCP or UDP session is encrypted or not. In the training phase the packages are used, since the characterization is much more refined than when the training is done per session.

Although it is true that the variables that measure uniformity are good variables, others that understand the behavior in the probability distribution must be included. For example, an unencrypted payload should have a majority of printable ASCII carateceres, although there is a distortion by the protocol headers of the application layer.


# Training phase
## Training data

The training data is made up of a traffic sample of BBVA Central Services buildings. These are in the form of a PCAP file, which occupies more than 4 GB.
### Description

In this phase, they use packages that use different protocols. To filter them, Wireshark filters are used (also applicable to TShark). All the packages used in the training have been analyzed thoroughly before. Including a false label in this phase will lead to bad results. It is not necessary to use too many examples, since Naive Bayes trains well with few data. The protocols used for training are the following:

- **TELNET**: not encrypted
- **SMB**: encrypted
- **SSH**: encrypted
- **SSL**: encrypted
- **HTTP**: not encrypted 

### PCAPS
In order to reproduce this study, the filters that were applied to the building traffic PCAP to obtain the training data are provided.

- **TELNET**: telnet.data
- **SSH**: ssh.encrypted_packet
- **SSL**: ssl and not ssl.handshake
- **HTTP**: http and not http2 and not ssl and not (tcp.srcport == 8080 or  tcp.srcport == 80 )

In the case of *HTTP*, filters are included to remove them from the building traffic sample. It can not be extrapolated to any PCAP.

To read the pcaps, you must clone the repository of <a href="https://github.com/dataEverything/file_processing"> file_processing </a> where there are functions that abstract the functionalities necessary for the execution of the algorithm contained in the present repository.

## Feature analysis
The following pre-processing has been carried out on the variables:
- Normalization between 0 and 1 to all the variables.
- Logarithmic transformation on: chi2, mode and most_ascii
- Exponential transformation over: entropy_cut

### Pearson chi-square test over the entire distribution

This test measures the discrepancy between an observed distribution and a uniform distribution, which would be the theoretical distribution in case the data were encrypted.

The higher the chi-square value, the less likely it is that a uniform distribution follows (that the null hypothesis is verified). In the same way, the closer the value of chi-square approaches zero, the more adjusted both distributions are.


In the encrypted packets, it is observed that chi2 is usually quite low. In the case of non-encrypted ones, there is a greater dispersion. This corroborates the hypothesis that the encrypted packets follow a uniform distribution.

<div align="center" >
  <img width="70%"  src="img/chi2.png"/>
</div>


### Shannon entropy over the entire distribution

Entropy measures the degree of uncertainty or homogeneity in information. Since a uniform distribution is homogeneous, entropy is an important feature to determine if data is encrypted.

The entropy in the encrypted packets is very high. In the unencrypted it is less high. The difference is not too marked by the problem of pairing with the headers.

<div align="center" >
  <img width="70%"  src="img/entropy.png"/>
</div>

### Shannon entropy over part of the distribution 

This variable measures the same as the previous one, but eliminates the following positions:

- First
- Second
- Last
- Mode (most repeated value once those mentioned above have been eliminated)

The reason why this variable exists is that PCAPS parsing tools do not have certain protocols of the application layer implemented and read their headers as if they were the payload. This causes peaks in encrypted packets that make the entropy decrease despite being encrypted the payload.


The entropy_cut variable differentiates the encrypted packets from those that are not because some encrypted ones saw their entropy diminished by the header of the layer 7 protocols, which the PCAPS parsing tools do not implement in some cases.

<div align="center" >
  <img width="70%"  src="img/entropy_cut.png"/>
</div>

### Mode
Mode is the most repeated byte. At times, it may serve to characterize the protocol. 

Mode has a higher concentration in high values in unencrypted packets.


<div align="center" >
  <img width="70%"  src="img/mode.png"/>
</div>

### Variety
The variety counts how many bytes there are in the distribution that are not zero.

The payload variety of a packet is the count of the unique bytes that there are.

<div align="center" >
  <img width="70%"  src="img/variety.png"/>
</div>


### Dispersion
The dispersion measures the width of the distribution, taking into account the zeros.

In the encrypted packets, the dispersion is usually maximum. Not only the bytes from 0 to 127 are used, but there are bytes in 0 and 255. In the non-encrypted ones, however, they tend to have a dispersion inside the printable ASCII except in case they are rich files, such as those of the Microsoft Office suite.

<div align="center" >
  <img width="70%"  src="img/dispersion.png"/>
</div>

### Most_ASCII
Certain bytes belong to the printable ASCII table. If these are used more than the rest, you can indicate that the data is not encrypted.

The variable most_ascii represents the proportion of printable ASCII characters with respect to the rest of the bytes that the payload of the package contains. In encryption, it is usually very low and higher in non-encrypted ones.

<div align="center" >
  <img width="70%"  src="img/most_ascii.png"/>
</div>

### Non_printable
It has been observed that some bytes within the distribution appear frequently in the encrypted data.

The non_printable variable represents the proportion of non-printable bytes with respect to the rest of the bytes that the package payload contains.

<div align="center" >
  <img width="70%"  src="img/non_printable.png"/>
</div>

### Average
The mean is calculated to detect biases in the distribution.

The average is centered on the case of encrypted packets. This happens in uniform distributions, which is a good sign.

<div align="center" >
  <img width="70%"  src="img/mean.png"/>
</div>

### Feature visualization

In general terms, you can see in the graphs the different characterization provided by the variables to the encrypted and unencrypted packets, thus being able to model their behavior.

A separation of the variables in the scatterplots matrix is observed. In blue are the unencrypted and in green the encrypted.

<div align="center" >
  <img width="70%"  src="img/distribution.png"/>
</div>

# Test phase
Once the variables have been studied, Naive Bayes is used as a classification algorithm. The trained model is saved for later use.

To apply the algorithm, the PCAPS that are contained in the specified route are read. This part has been separated from the rest of the algorithm in order to change the way PCAPS is read, so Kafka could be used instead of reading the files of a directory.

After applying Wireshark filters, a package is selected that is known to be encrypted or not and the session that contains it is extracted. Then the session is re-analyzed, exploring the headers and / or visually to ensure that the provided label is correct. A list with the expected labels is constructed, given the PCAPS that are provided for the test phase.

Data contained in the test PCAPS:
- **Sessions**: there are TCP and UDP sessions in the PCAPS.
- **Packages with several files**: there are sessions containing non-encrypted files transferred by SMB (not to be confused with SMB3). It is known that these are not encrypted because they can be downloaded in Wireshark with the option *Export Objects / SMB *


Then the payload is extracted from each PCAP giving a result of file per session. That is to say, that each file is labeled instead of labeling a package, as was done in the training phase, since the objective is to know if a session is encrypted or not. To facilitate the labeling, encrypted PCAPS are saved in different folders of those that are not.

The pre-processing in both cases is the same, so the algorithm could be applied with unlabelled sessions.

## Naive Bayes

It is a probabilistic classifier based on Bayes' theorem and some additional simplifying hypotheses. The parameters used to train this classifier are the characteristics of the distributions of the training set.

In the *NetworkSessionClassification* notebook you can see how the model is applied to each package, and the result is summarized per session.

## Algorithm

After training the algorithm that detects whether a packet is encrypted or not, you should find a way to give a result per session. To fulfill this purpose, a class that is potentially deployable in production is developed, which is found in the *NetworkSessionClassification* notebook.

The labels of the packages are -1 and 1 to be able to know how encrypted or unencrypted it is, since if the labels of the packages were 0 and 1, one could only know how encrypted it is.

When answering if a session is encrypted or not, the following values are collected:

- **Vector P**: classification of the package within a session. That will be -1 in the unencrypted packets and 1 in the encrypted ones
- **Vector L**: size of each packet within a session

The product of both vectors is added and the result is normalized with a hyperbolic tangent.

<div align="center" >
  <img width="70%"  src="img/tangente_hiperbolica.PNG"/>
</div>


The formula that is used to give a result per session is the following:

<div align="center" >
  <img width="30%"  src="img/formula_cifrado.png"/>
</div>


Where:
- *n* is the number of packages that a session contains
- The vector *P* is the vector that contains the size of each packet within the session in bytes.
- The vector *L* is the vector that contains the classification result, which will be 1 or -1.

The prediction will be a value between 1 and -1, since it is normalized by a hyperbolic tangent.
- Not encrypted: -1
- Encryption: 1
- Unknown: 0

## Execution of the algorithm

When the algorithm is applied, the function that returns a list of routes is called. Each file must contain a single session. This part can be replaced by any other type of entry, as long as a raw PCAP is passed to the class.

Once, you are given the path where the session to be analyzed is located, the function that reads the file is called and it is passed to the class **SessionClassification** and the results are saved in the list *y_pred* to contrast them with those expected in order to measure the effectiveness of the algorithm. This list will have one element per session.

To give a result per session, multiply the list with the predictions and the list of the sizes of the packages that belong to a session. This is done because there are cases in which a session is not encrypted from the beginning. For example, an encrypted TCP session may have a pre-negotiation phase that will not be. However, that session should be classified as encrypted. Normally, negotiation packages are smaller than encrypted data, so size acts as a weight that models encryption in a session.

## Data used for the test phase
The PCAPS used for the test come from the latter part of PCAP sample traffic and traffic buildings captured during the Dementors from Hogwarts' platform.

# Results

Although there are false positives and false negatives when classifying the packages, these errors occur when the packages are very small, so these errors are corrected when a session is taken into account as a whole.

## Model metrics
### Confusion Matrix

The following metrics are calculated:
- TNR: 0.97
- TPR: 1.00
- Precision: 0.92
- Recall: 1.00
- F-Measure: 0.96

The matrix of confusion is shown below.

<div align="center" >
  <img width="70%"  src="img/confusion_matrix.png"/>
</div>

### ROC curve

Curve Receiver Operating Characteristic is a graphical representation of the sensitivity versus specificity for a binary classifier system as is the case we found, according to the discrimination threshold is varied. The area under the ROC curve is 0.98.

<div align="center">
  <img width="70%"  src="img/roc_curve.png"/>
</div>

# Conclusion 

It is possible, from a PCAP or raw network traffic, to know if a session is encrypted or not although not all protocols are implemented by the available PCAPS parsing tools, thus being able to classify any session without any pre-filter This would allow to know the habitual behavior of ports, servers, applications and other groupings that are wished to realize. It could thus detect anomalies that could trigger security alerts in the future.

# Next steps
## Apply the model in streaming
When it comes to training and testing the model, it does not make sense to apply it in real time. It has been tried to apply the model in streaming with Kafka, obtaining good results.

## Filter of sessions without payload at the level of the application layer
False positives are given in packets that only contain protocol headers, such as a *Create Request File* of SMB2. With a better protocol pairing or with an additional filter, the effectiveness of the model could be improved.

## Characterization by protocol
A study could be made within each protocol of the percentage of encrypted traffic that receives and analyze variations on it. Another option is to add more variables describing the protocol itself, creating a module parallel or subsequent to the encryption algorithm.
