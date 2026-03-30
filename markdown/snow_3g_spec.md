\section*{Specification of the 3GPP Confidentiality and \\ Integrity Algorithms UEA2 \& UIA2. Document 1: UEA2 and UIA2 Specification}

\begin{tabular}{|l|l|l|}
\hline \multicolumn{3}{|c|}{Document History} \\
\hline V1.0 & $10^{\text {th }}$ January 2006 & Publication \\
\hline V1.1 & $6^{\text {th }}$ September 2006 & No change to the algorithm specification at all, just removal of an unwanted page header \\
\hline
\end{tabular}

\section*{PREFACE}

This specification has been prepared by the 3GPP Task Force, and gives a detailed specification of the 3GPP confidentiality algorithm UEA2 and the 3GPP integrity algorithm UIA2.

This document is the first of four, which between them form the entire specification of 3GPP Confidentiality and Integrity Algorithms:
- Specification of the 3GPP Confidentiality and Integrity Algorithms UEA2 \& UIA2.
Document 1: UEA2 and UIA2 Algorithm Specifications.
- Specification of the 3GPP Confidentiality and Integrity Algorithms UEA2 \& UIA2.
Document 2: SNOW 3G Algorithm Specification.
- Specification of the 3GPP Encryption and Confidentiality Algorithms UEA2 \& UIA2.
Document 3: Implementors' Test Data.
- Specification of the 3GPP Encryption and Confidentiality Algorithms UEA2 \& UIA2.
Document 4: Design Conformance Test Data.
The normative part of the specification of the UEA2 (confidentiality) and UIA2 (integrity) algorithms is in the main body of this document. The annexes to this document are purely informative.

The informative section of this document includes four informative annexes: Annex 1 contains remarks about the mathematical background of some functions of UIA2. Annex 2 contains implementation options for some functions of UIA2. Annex 3 contains illustrations of functional elements of the algorithms, while Annex 4 contains an implementation program listing of the cryptographic algorithm specified in the main body of this document, written in the programming language C .

The normative section of the specification of the stream cipher (SNOW 3G) on which they are based is in the main body of Document 2. The annexes to that document, and Documents 3 and 4 above, are purely informative.

\author{
Blank Page
}

\section*{TABLE OF CONTENTS}
1. OUTLINE OF THE NORMATIVE PART ..... 8
2. INTRODUCTORY INFORMATION ..... 8
2.1. Introduction ..... 8
2.2. Notation ..... 8
2.3. List of Variables ..... 10
3. CONFIDENTIALITY ALGORITHM UEA2 ..... 11
3.1. Introduction ..... 11
3.2. Inputs and Outputs ..... 11
3.3. Components and Architecture ..... 11
3.4. Initialisation ..... 11
3.5. Keystream Generation ..... 12
3.6. Encryption/Decryption ..... 12
4. INTEGRITY ALGORITHM UIA2 ..... 13
4.1. Introduction ..... 13
4.2. Inputs and Outputs ..... 13
4.3. Components and Architecture ..... 13
4.4. Initialisation ..... 14
4.5. Calculation ..... 15
ANNEX 1 Remarks about the mathematical background of some operations of the UIA2 Algorithm ..... 17
1.1. The function EVAL_M ..... 17
1.2. The function $\operatorname{MUL}(\mathrm{V}, \mathrm{P}, \mathrm{c})$ ..... 17
ANNEX 2 Implementation options for some operations of the UIA2 Algorithm ..... 18
2.1. Procedure Pre_Mul_P ..... 18
2.2. Function Mul_P ..... 18
ANNEX 3 Figures of the UEA2 and UIA2 Algorithms ..... 20
ANNEX 4 Simulation Program Listing ..... 23
4.1. UEAII ..... 23
4.2. UIAII ..... 24

\section*{REFERENCES}
[1] 3rd Generation Partnership Project; Technical Specification Group Services and System Aspects; 3G Security; Security Architecture (3G TS 33.102 version 6.3.0).
[2] 3rd Generation Partnership Project; Technical Specification Group Services and System Aspects; 3G Security; Cryptographic Algorithm Requirements; (3G TS 33.105 version 6.0.0).
[3] Specification of the 3GPP Confidentiality and Integrity Algorithms; Document 1: $\boldsymbol{f} \mathbf{8}$ and $\boldsymbol{f} \mathbf{9}$ specifications; (3GPP TS35.201 Release 6).
[4] Specification of the 3GPP Confidentiality and Integrity Algorithms UEA2 \& UIA2. Document 1: UEA2 and UIA2 specifications.
[5] Specification of the 3GPP Confidentiality and Integrity Algorithms UEA2 \& UIA2. Document 2: SNOW 3G specification.
[6] Specification of the 3GPP Confidentiality and Integrity Algorithms UEA2 \& UIA2. Document 3: Implementors' Test Data.
[7] Specification of the 3GPP Confidentiality and Integrity Algorithms UEA2 \& UIA2. Document 4: Design Conformance Test Data.
[8] P. Ekdahl and T. Johansson, "A new version of the stream cipher SNOW", in Selected Areas in Cryptology (SAC 2002), LNCS 2595, pp. 47-61, SpringerVerlag.

\section*{NORMATIVE SECTION}

This part of the document contains the normative specification of the Confidentiality and Integrity algorithms.

\section*{1. OUTLINE OF THE NORMATIVE PART}

Section 2 introduces the algorithm and describes the notation used in the subsequent sections.
Section 3 specifies the confidentiality algorithm UEA2.
Section 4 specifies the integrity algorithm UIA2.

\section*{2. INTRODUCTORY INFORMATION}

\subsection*{2.1. Introduction}

Within the security architecture of the 3GPP system there are standardised algorithms for confidentiality (f8) and integrity (f9). A first set of algorithms for f8 and f9 (UEA1 and UIA1) has already been specified [3]. A second set of algorithms for $\boldsymbol{f} \mathbf{8}$ and $\boldsymbol{f} \mathbf{9 ( U E A 2}$ and UIA2) are fully specified here: The second set of these algorithms is based on the SNOW 3G algorithm that is specified in a companion document [5].

The confidentiality algorithm UEA2 is a stream cipher that is used to encrypt/decrypt blocks of data under a confidentiality key CK. The block of data may be between 1 and 20000 bits long. The algorithm uses SNOW 3G as a keystream generator

The integrity algorithm UIA2 computes a 32-bit MAC (Message Authentication Code) of a given input message using an integrity key IK. The approach adopted uses SNOW 3G.

\subsection*{2.2 Notation}

\subsection*{2.2.1. Radix}

We use the prefix $\mathbf{0 x}$ to indicate hexadecimal numbers.

\subsection*{2.2.2. Conventions}

We use the assignment operator ' $=$ ', as used in several programming languages. When we write
<variable> = <expression>
we mean that <variable> assumes the value that <expression> had before the assignment took place. For instance,
$$
x=x+y+3
$$
means
$($ new value of x$)$ becomes $($ old value of x$)+($ old value of y$)+3$.

\subsection*{2.2.3. Bit/Byte ordering}

All data variables in this specification are presented with the most significant bit (or byte) on the left hand side and the least significant bit (or byte) on the right hand side. Where a variable is broken down into a number of sub-strings, the left most (most significant) substring is numbered 0 , the next most significant is numbered 1 and so on through to the least significant.

For example an n -bit $\mathbf{M E S S A G E}$ is subdivided into 64-bit substrings $\mathbf{M B}_{\mathbf{0}}, \mathbf{M B}_{\mathbf{1}}, \mathbf{M B}_{\mathbf{2}}, \ldots$. So if we have a message:

0x0123456789ABCDEFFEDCBA98765432108654381AB594FC28786404C50A37...
we have:
$$
\begin{aligned}
& \mathbf{M B}_{0}=0 \times 0123456789 \mathrm{ABCDEF} \\
& \mathbf{M B}_{1}=0 \mathrm{xFEDCBA} 9876543210 \\
& \mathbf{M B}_{2}=0 \mathrm{x} 86545381 \mathrm{AB} 594 \mathrm{FC} 2 \\
& \mathbf{M B}_{3}=0 \mathrm{x} 8786404 \mathrm{C} 50 \mathrm{~A} 37 \ldots
\end{aligned}
$$

In binary this would be:
$$
\begin{aligned}
& 000000010010001101000101011001111000100110101011110011011110111111111110 \\
& \text { with } \mathbf{M B}_{0}=0000000100100011010001010110011110001001101010111100110111101111 \\
& \mathbf{M B}_{1}=1111111011011100101110101001100001110110010101000011001000010000 \\
& \mathbf{M B}_{2}=1000011001010100010100111000000110101011010110010100111111000010 \\
& \mathbf{M B}_{3}=1000011110000110010000000100110001010000101000110111 \ldots
\end{aligned}
$$

\subsection*{2.2.4. List of Symbols}
$=\quad$ The assignment operator
⊕ The bitwise exclusive-OR operation
|| The concatenation of the two operands.
$\lceil x\rceil$ The smallest integer greater than or equal to the real number $x$.
$\&_{\mathrm{n}} \quad$ The bitwise AND operation in an n -bit register.
$\ll_{n} \mathrm{t} \mathrm{t}$-bit left shift in an n -bit register.
$\gg_{n} \mathrm{t} \mathrm{t}$-bit right shift in an n -bit register

\subsection*{2.3. List of Variables}

\begin{tabular}{|l|l|}
\hline BEARER & the 5-bit input to the UEA2 function. \\
\hline CK & the 128-bit confidentiality key. \\
\hline COUNT & the 32-bit time variant input to the UEA2 and UIA2 functions (COUNTC for UEA2 and COUNT-I for UIA2) \\
\hline DIRECTION & the 1-bit input to both the UEA2 and UIA2 functions indicating the direction of transmission (uplink or downlink). \\
\hline FRESH & the 32-bit random input to the UIA2 function. \\
\hline IBS & the input bit stream to the UEA2 function. \\
\hline IK & the 128-bit integrity key. \\
\hline KS[i] & the $i^{\text {th }}$ bit of keystream produced by the keystream generator. \\
\hline LENGTH & the input to the UEA2 and UIA2 functions which specifies the number of bits in the input bitstream (1-20000). \\
\hline MAC-I & the 32-bit message authentication code (MAC) produced by the integrity function UIA2. \\
\hline MESSAGE & the input bitstream of LENGTH bits that is to be processed by the UIA2 function. \\
\hline OBS & the output bit stream from the UEA2 function. \\
\hline $\mathrm{Z}_{1}, \mathrm{Z}_{2}, \ldots$ & the 32-bit words forming the keystream sequence of SNOW 3G. The word produced first is $\mathrm{z}_{1}$, the next word $\mathrm{z}_{2}$ and so on. \\
\hline
\end{tabular}

\section*{3. CONFIDENTIALITY ALGORITHM UEA2}

\subsection*{3.1. Introduction}

The confidentiality algorithm UEA2 is a stream cipher that encrypts/decrypts blocks of data between 1 and 20000 bits in length.

\subsection*{3.2. Inputs and Outputs}

The inputs to the algorithm are given in Table 1, the output in Table 2:

\begin{table}
\begin{tabular}{|l|l|l|}
\hline Parameter & Size (bits) & Comment \\
\hline COUNT-C & 32 & Frame dependent input COUNT-C[0]...COUNT-C[31] \\
\hline BEARER & 5 & Bearer identity BEARER[0]...BEARER[4] \\
\hline DIRECTION & 1 & Direction of transmission DIRECTION[0] \\
\hline CK & 128 & Confidentiality key CK[0]....CK[127] \\
\hline LENGTH & 16 & The number of bits to be encrypted/decrypted \\
\hline IBS & LENGTH & Input bit stream IBS[0]....IBS[LENGTH-1] \\
\hline
\end{tabular}
\captionsetup{labelformat=empty}
\caption{Table 1. UEA2 inputs}
\end{table}

\begin{table}
\begin{tabular}{|l|c|l|}
\hline Parameter & Size (bits) & Comment \\
\hline OBS & LENGTH & \begin{tabular}{l} 
Output bit stream \\
OBS[0]....OBS[LENGTH-1]
\end{tabular} \\
\hline
\end{tabular}
\captionsetup{labelformat=empty}
\caption{Table 2. UEA2 output}
\end{table}

\subsection*{3.3. Components and Architecture}

The keystream generator is based on SNOW 3G that is specified in [5]. SNOW 3G is a word oriented stream cipher and generates a keystream in multiples of 32-bits.

\subsection*{3.4. Initialisation}

In this section we define how the keystream generator is initialised with the key variables before the generation of keystream bits.

All variables have length 32 and are presented with the most significant bit on the left hand side and the least significant bit on the right hand side.
$\mathbf{K}_{\mathbf{3}}=\mathbf{C K [ 0 ]}\|\mathbf{C K [ 1 ]}\| \mathbf{C K [ 2 ]}\|\ldots\| \mathbf{C K [ 3 1 ]}$
$\mathrm{K}_{2}=\mathrm{CK}[32]\|\mathrm{CK}[33]\| \mathrm{CK}[34]\|\ldots\| \mathrm{CK}[63]$
$\mathbf{K}_{1}=\mathbf{C K}[\mathbf{6 4}]\|\mathbf{C K}[\mathbf{6 5}]\| \mathbf{~ C K [ 6 6 ]}\|\ldots\| \mathbf{~ C K [ 9 5 ]}$
$\mathbf{K}_{\mathbf{0}} \boldsymbol{=} \mathbf{C K [ 9 6 ]} \boldsymbol{\| \mathbf { C K } [ 9 7 ] \| \mathbf { C K [ 9 8 ] } \| \boldsymbol { \ldots } \boldsymbol { \| } \boldsymbol { | } \mathbf { C K [ 1 2 7 ] }}$
IV $_{3}=$ COUNT-C[0] || COUNT-C[1] || COUNT-C[2] || ... || COUNT-C[31]
$\mathrm{IV}_{2}=$ BEARER[0] $\|$ BEARER[1] $\|\ldots\|$ BEARER[4] $\|$ DIRECTION[0] $\|0\| \ldots \| 0$
IV ${ }_{1}=$ COUNT-C[0] || COUNT-C[1] || COUNT-C[2] || ... || COUNT-C[31]
$\mathrm{IV}_{0}=$ BEARER[0] || BEARER[1] | $\ldots|\mid$ BEARER[4] || DIRECTION[0] | 0$||\ldots| \mid 0$
SNOW 3G is initialised as described in document [5].

\subsection*{3.5. Keystream Generation}

Set $\mathbf{L}=\lceil$ LENGTH $/ 32\rceil$.
SNOW 3G is run as described in document [5] to produce the keystream consisting of the 32bit words $\mathbf{z}_{1} \ldots \mathbf{z}_{\mathrm{L}}$. The word produced first is $\mathbf{z}_{1}$, the next word $\mathbf{z}_{2}$ and so on.

The sequence of keystream bits is $\mathbf{K S [ 0 ]} .$. KS[LENGTH-1], where KS[0] is the most significant bit and KS[31] is the least significant bit of $\mathbf{z}_{1}, \mathbf{K S [ 3 2 ]}$ is the most significant bit of $\mathbf{z}_{2}$ and so on.

\subsection*{3.6. Encryption/Decryption}

Encryption/decryption operations are identical operations and are performed by the exclusiveOR of the input data (IBS) with the generated keystream (KS).

For each integer $\boldsymbol{i}$ with $0 \leq \boldsymbol{i} \leq \mathbf{L E N G T H}-1$ we define:
$\mathbf{O B S}[i]=\mathbf{I B S}[i] \oplus \mathbf{K S}[i]$.

\section*{4. INTEGRITY ALGORITHM UIA2}

\subsection*{4.1. Introduction}

The integrity algorithm UIA2 computes a Message Authentication Code (MAC) on an input message under an integrity key $\boldsymbol{I} \boldsymbol{K}$. The message may be between 1 and 20000 bits in length.

For ease of implementation the algorithm is based on the same stream cipher (SNOW 3G) as is used by the confidentiality algorithm UEA2.

\subsection*{4.2. Inputs and Outputs}

The inputs to the algorithm are given in table 3 , the output in table 4 :

\begin{table}
\begin{tabular}{|l|l|l|}
\hline Parameter & Size (bits) & Comment \\
\hline COUNT-I & 32 & Frame dependent input COUNT-I[0]…COUNTI[31] \\
\hline FRESH & 32 & Random number FRESH[0]...FRESH[31] \\
\hline DIRECTION & 1 & Direction of transmission DIRECTION[0] \\
\hline IK & 128 & Integrity key IK[0]...IK[127] \\
\hline LENGTH & 64 & The number of bits to be 'MAC'd \\
\hline MESSAGE & LENGTH & Input bit stream \\
\hline
\end{tabular}
\captionsetup{labelformat=empty}
\caption{Table 3. UIA2 inputs}
\end{table}

\begin{table}
\begin{tabular}{|l|r|l|}
\hline Parameter & Size (bits) & Comment \\
\hline MAC-I & 32 & \begin{tabular}{l} 
Message authentication code MAC-I[0]...MAC- \\
I[31]
\end{tabular} \\
\hline
\end{tabular}
\captionsetup{labelformat=empty}
\caption{Table 4. UIA2 output}
\end{table}

\subsection*{4.3. Components and Architecture}

\subsection*{4.3.1. SNOW 3G}

The integrity function uses SNOW 3G that is specified in [5]. SNOW 3G is a word oriented stream cipher and generates from the key and an initialisation variable five 32-bit-words $\mathbf{z}_{1}, \mathbf{z}_{2}$, $\mathbf{Z}_{3}, \mathbf{z}_{4}$ and $\mathbf{z}_{5}$.

\subsection*{4.3.2. MULx}

MULx maps 128 bits to 64 bits. Let $V$ and $c$ be 64-bit input values. Then MULx is defined: If the leftmost (i.e. the most significant) bit of $V$ equals 1 , then
$$
\begin{aligned}
& \operatorname{MULx}(V, c)=\left(V \ll_{64} 1\right) \oplus c, \\
& \text { else } \\
& \operatorname{MULx}(V, c)=V \ll_{64} 1 .
\end{aligned}
$$

\subsection*{4.3.3. MULxPOW}

MULxPOW maps 128 bits and a positive integer $i$ to 64 bit. Let $V$ and $c$ be 64-bit input values, then $\mathrm{MULxPOW}(V, i, c)$ is recursively defined:
If $i$ equals 0 , then
$$
\operatorname{MULxPOW}(V, i, c)=V,
$$
else
$$
\operatorname{MULxPOW}(V, i, c)=\operatorname{MULx}(\operatorname{MULxPOW}(V, i-1, c), c) .
$$
4.3.4. MUL

MUL maps 192 bits to 64 bit. Let $V, P$ and $c$ be 64-bit input values.
Then the 64-bit output result of $\operatorname{MUL}(V, P, c)$ is computed as follows:
- result $=0$.
- for $i=0$ to 63 inclusive
- if $\left(P \gg_{64} i\right) \&_{64} 0 \times 01$ equals $0 \times 01$, then result $=$ result $\oplus \operatorname{MULxPOW}(V, i, c)$.

\subsection*{4.4. Initialisation}

In this section we define how the keystream generator is initialised with the key and initialisation variables before the generation of keystream bits.

All variables have length 32 bits and are presented with the most significant bit on the left hand side and the least significant bit on the right hand side.
$\left.\mathbf{K}_{\mathbf{3}}=\mathbf{I K}[0]\|\mathbf{I K}[1]\| \mathbf{I K}[2] \| \mathbf{n}\right] \| \mathbf{I K}[31]$
$\mathbf{K}_{\mathbf{2}}=\mathbf{I K}[32]| | \mathbf{I K}[33]| | \mathbf{I K}[34]| |$... $|\mid \mathbf{I K}[63]$
$\mathbf{K}_{\mathbf{1}}=\mathbf{I K}[64]| | \mathbf{I K}[65]| | \mathbf{I K}[66]| |$... $|\mid \mathbf{I K}[95]$
$\mathbf{K}_{\mathbf{0}}=\mathbf{I K}[96]| | \mathbf{I K}[97]| | \mathbf{I K}[98]| |$ … $|\mid$ IK[127]
IV $_{3}=$ COUNT-I[0] || COUNT-I[1] || COUNT-I[2] || ... || COUNT-I[31]
$\mathbf{I V}_{2}=\mathbf{F R E S H}[0]\|\mathbf{F R E S H}[1]\| \mathbf{F R E S H}[2]\|\ldots\|$ FRESH[31]
$\mathbf{I V}_{\mathbf{1}} \boldsymbol{=}$ DIRECTION[0] ⊕ COUNT-I[0] || COUNT-I[1] || COUNT-I[2] || ... || COUNT-I[31]
$\mathbf{I V}_{\mathbf{0}}=\mathbf{F R E S H}[0]\|\mathbf{F R E S H}[1]\| \ldots\|\mathbf{F R E S H}[15]\| \mathbf{F R E S H}[16] \oplus \mathbf{D I R E C T I O N}[0]\|\mathbf{F R E S H}[17]\| \ldots \|$ FRESH $[31]$

SNOW 3G is initialised as described in document [5].

\subsection*{4.5. Calculation}

Set $\mathbf{D}=\lceil$ LENGTH $/ 64\rceil+1$.
SNOW 3G is run as described in document [5] in order to produce 5 keystream words $\mathbf{z}_{1}, \mathbf{z}_{2}$, $\mathbf{Z}_{3}, \mathbf{Z}_{4}, \mathbf{z}_{5}$.

Set $\quad \mathbf{P}=\mathbf{z}_{1} \| \mathbf{z}_{2}$
and $\mathbf{Q}=\mathbf{z}_{3} \| \mathbf{z}_{4}$.
Let OTP[0], OTP[1], OTP[2], ..., OTP[31] be bit-variables such that
$$
\mathbf{z}_{5}=\mathbf{O T P}[0]\|\mathbf{O T P}[1]\| \ldots \| \mathbf{O T P}[31]
$$
i.e. OTP $[0]$ is the most and OTP $[31]$ the least significant bit of $\mathbf{z}_{5}$.
For $0 \leq i \leq \mathbf{D}-3$ set
$$
\mathbf{M}_{i}=\mathbf{M E S S A G E}[64 i] \| \text { MESSAGE }[64 i+1]\|. . .\| \text { MESSAGE }[64 i+63] .
$$

Set
$$
\mathbf{M}_{\mathbf{D}-2}=\text { MESSAGE[64(D-2)] }\|\ldots\| \text { MESSAGE[LENGTH-1] } \| 0 \ldots 0 .
$$

Let LENGTH[0], LENGTH[1], ..., LENGTH[63] be the bits of the 64-bit representation of LENGTH, where LENGTH[0] is the most and LENGTH[63] is the least significant bit.

Set $\mathbf{M}_{\mathbf{D}-1}=\mathbf{L E N G T H}[0]\|\mathbf{L E N G T H}[1]\| \ldots \|$ LENGTH[63].
Compute the function Eval_M:
- Set the 64-bit variable EVAL $=0$.
- for $i=0$ to $\mathbf{D}-2$ inclusive:

○ $\mathbf{E V A L}=\operatorname{Mul}\left(\mathbf{E V A L} \oplus \mathbf{M}_{i}, \mathbf{P}\right.$, 0x000000000000001b ).
Set EVAL $=\mathbf{E V A L} \oplus \mathbf{M}_{D-1}$
Now we multiply EVAL by Q:
$$
\mathbf{E V A L}=\operatorname{Mul}(\mathbf{E V A L}, \mathbf{Q}, 0 \times 000000000000001 \mathrm{~b})
$$

Let $\mathbf{E V A L}=\mathbf{e}_{0}\left\|\mathbf{e}_{1}\right\| \ldots \| \mathbf{e}_{63}$ with $\mathbf{e}_{0}$ the most and $\mathbf{e}_{63}$ the least significant bit.
For $0 \leq i \leq 31$, set
$$
\mathbf{M A C}-\mathbf{I}[i]=\mathbf{e}_{i} \oplus \mathbf{O T P}[i]
$$

The bits $\mathbf{e}_{32}, \ldots, \mathbf{e}_{63}$ are discarded.

\section*{INFORMATIVE SECTION}

This part of the document is purely informative and does not form part of the normative specification of the Confidentiality and Integrity algorithms.

\section*{ANNEX 1 \\ Remarks about the mathematical background of some operations of the UIA2 Algorithm}

\subsection*{1.1. The function EVAL_M}

The first part (the function EVAL_M) of the calculations for the UIA2 algorithm corresponds to the evaluation of a polynomial at a secret point: From the bits and the length of MESSAGE a polynomial $\mathbf{M} \in \mathrm{GF}\left(2^{64}\right)[X]$ is defined. This polynomial is evaluated at the point $\mathbf{P} \in \mathrm{GF}\left(2^{64}\right)$ defined by $\mathbf{z}_{1} \| \mathbf{z}_{2}$.

This can be seen as follows:
Consider the Galois Field $\mathrm{GF}\left(2^{64}\right)$ where elements of the field are represented as polynomials over $\mathrm{GF}(2)$ modulo the irreducible polynomial $x^{64}+x^{4}+x^{3}+x+1$.

Variables consisting of 64 bits can be mapped to this field by interpreting the bits as the coefficients of the corresponding polynomial.

For example for $0 \leq i \leq \mathbf{D}$-3 the variable
$\mathbf{M}_{i}=\mathbf{M E S S A G E}[64 i]\|\mathbf{M E S S A G E}[64 i+1]\| . . . \|$ MESSAGE[64i+62] $\|$ MESSAGE[64i+63] is interpreted as
MESSAGE $[64 i] x^{63}+$ MESSAGE $[64 i+1] x^{62}+\ldots+$ MESSAGE $[64 i+62] x+$ MESSAGE[64i+63].

Construct the polynomial $\mathbf{M}$ of degree $\mathbf{D}$ - 1 in $\mathrm{GF}\left(2^{64}\right)[X]$ as
$\mathbf{M}(X)=\mathbf{M}_{\mathbf{0}} X^{\mathbf{D}-1}+\mathbf{M}_{\mathbf{1}} X^{\mathbf{D}-\mathbf{2}}+\ldots+\mathbf{M}_{\mathbf{D}-\mathbf{2}} X+\mathbf{M}_{\mathbf{D}-\mathbf{1}}$.
Evaluate the polynomial $\mathbf{M}$ at the point $\mathbf{P}$, i.e. compute
$\left.\mathbf{M}(\mathbf{P})=\mathbf{M}_{0} \mathbf{P}^{\mathbf{D}-1}+\mathbf{M}_{1} \mathbf{P}^{\mathbf{D}-2}+\ldots+\mathbf{M}_{\mathbf{D}-2} \mathbf{P}+\mathbf{M}_{\mathbf{D}-1}=\left(\ldots\left(\mathbf{M}_{0} \mathbf{P}+\mathbf{M}_{1}\right) \mathbf{P}+\mathbf{M}_{2}\right) \mathbf{P}+\ldots+\mathbf{M}_{\mathbf{D}-2}\right) \mathbf{P}+ \mathbf{M}_{\mathbf{D}-\mathbf{1}}$.

This is done in the function Eval_M in 4.5.

\subsection*{1.2 The function $\operatorname{MUL}(\mathrm{V}, \mathrm{P}, \mathrm{c})$}

The function $\operatorname{MUL}(V, P, \mathrm{c})$ (see 4.3.4) corresponds to a multiplication of $V$ by $P$ in $\mathrm{GF}\left(2^{64}\right)$. Here $\mathrm{GF}\left(2^{64}\right)$ is described as $\mathrm{GF}(2)(\beta)$ where $\beta$ is a root of the $\mathrm{GF}(2)[\mathrm{x}]$ polynomial $\mathrm{x}^{64}+ \mathrm{c}_{0} \mathrm{x}^{63}+\ldots+\mathrm{c}_{62} \mathrm{x}+\mathrm{c}_{63}$ and $\mathrm{c}=\mathrm{c}_{0}\left\|\mathrm{c}_{1}\right\| \ldots \| \mathrm{c}_{63}$.

\section*{ANNEX 2 \\ Implementation options for some operations of the UIA2 Algorithm}

The function MUL (see 4.3.4) can be implemented using table lookups. This might accelerate execution of the function EVAL_M, as for the evaluation of the polynomial only multiplication by a constant factor $P$ is needed.

There are different possible sizes for the tables. Here we use 8 tables with 256 entries, but for example it is also possible to use 16 tables with 16 entries.

In order to execute MUL by table-lookups first Pre_Mul_P (see 2.1) is executed, which generates the tables. Then in MUL_P (see 2.2) the multiplication is performed by 8 tablelookups and an xor of the results.

Hence in 4.5 instead of $\mathbf{E V A L}=\operatorname{Mul}\left(\mathrm{EVAL} \oplus \mathbf{M}_{i}, \mathbf{P}\right.$, 0x1b ) we can use $\mathbf{E V A L}=$ Mul_P $\left(\mathbf{E V A L} \oplus \mathbf{M}_{i}\right)$.

\subsection*{2.1. Procedure Pre_Mul_P}

In order to be able to compute Mul_P (see 2.2) the procedure Pre_Mul_P is executed once before the first call of Mul_P.
Pre_Mul_P computes from the 64-bit input $\mathbf{P}$ eight tables $\mathrm{PM}[0], \mathrm{PM}[1], \ldots, \mathrm{PM}[7]$. Each of these tables contains 256 entries PM[j][0], PM[j][1], ..., PM[j][255] with 64 bits.

For $0 \leq \mathrm{j} \leq 7$ and $0 \leq X \leq 255$ the value $\mathrm{PM}[\mathrm{j}][X]$ corresponds to $X \mathbf{P} \mathrm{x}^{8 \mathrm{j}}$.
Let $r$ be the 64-bit value 0x000000000000001b.
- The tables are computed as follows:
$\mathrm{PM}[0][0]=\mathrm{PM}[1][0]=\mathrm{PM}[2][0]=\mathrm{PM}[3][0]=\mathrm{PM}[4][0]=\mathrm{PM}[5][0]=\mathrm{PM}[6][0]= \mathrm{PM}[7][0]=0$.
- $\mathrm{PM}[0][1]=\mathrm{P}$.
- for $i=1$ to 63 inclusive:
$\circ \mathrm{PM}\left[i \gg_{8} 3\right]\left[1 \ll_{8}\left(i \&_{8} 0 \mathrm{x} 07\right)\right]=\mathrm{PM}\left[(i-1) \gg_{8} 3\right]\left[1 \ll_{8}\left((i-1) \&_{8} 0 \mathrm{x} 07\right)\right] \ll_{64} 1$.
- if the leftmost bit of $\mathrm{PM}\left[(i-1) \gg_{8} 3\right]\left[1 \ll\left((i-1) \&_{8} 0 x 07\right)\right]$ equals 1 , then
$\mathrm{PM}\left[i \gg_{8} 3\right]\left[1 \ll_{8}\left(i \&_{8} 0 \mathrm{x} 07\right)\right]=\mathrm{PM}\left[i \gg_{8} 3\right]\left[1 \ll\left(i \&_{8} 0 \mathrm{x} 07\right)\right] \oplus r$.
- for $i=0$ to 7 inclusive
- for $j=1$ to 7 inclusive
- for $k=1$ to $\left(1 \ll_{8} j\right)-1$ inclusive
- $\mathrm{PM}[i]\left[\left(1 \ll_{8} j\right)+k\right]=\mathrm{PM}[i]\left[1 \ll_{8} j\right] \oplus \mathrm{PM}[i][k]$.

\subsection*{2.2 Function Mul_P}

The function Mul_P maps a 64-bit input $X$ to a 64-bit output.

Let $X=X_{0}\left\|X_{1}\right\| X_{2}\left\|X_{3}\right\| X_{4}\left\|X_{5}\right\| X_{6} \| X_{7}$, with $X_{0}$ the most and $X_{7}$ the least significant byte.
Compute Mul_ $\mathrm{P}(X)$ as
$\operatorname{Mul\_ P}(X)=\mathrm{PM}[0]\left[X_{7}\right] \oplus \mathrm{PM}[1]\left[X_{6}\right] \oplus \mathrm{PM}[2]\left[X_{5}\right] \oplus \mathrm{PM}[3]\left[X_{4}\right] \oplus \mathrm{PM}[4]\left[X_{3}\right] \oplus \mathrm{PM}[5]\left[X_{2}\right] \oplus \mathrm{PM}[6]\left[X_{1}\right] \oplus \mathrm{PM}[7]\left[\mathrm{X}_{0}\right]$.

\section*{ANNEX 3 \\ Figures of the UEA2 and UIA2 Algorithms}

\begin{figure}
\includegraphics[alt={},max width=\textwidth]{https://cdn.mathpix.com/cropped/385f8b5b-fd9b-4f2d-b7fd-b443bf1750cf-20.jpg?height=840&width=1762&top_left_y=826&top_left_x=146}
\captionsetup{labelformat=empty}
\caption{Figure 1: UEA2 Keystream Generator}
\end{figure}

\begin{figure}
\includegraphics[alt={},max width=\textwidth]{https://cdn.mathpix.com/cropped/385f8b5b-fd9b-4f2d-b7fd-b443bf1750cf-21.jpg?height=992&width=1630&top_left_y=242&top_left_x=406}
\captionsetup{labelformat=empty}
\caption{Figure 2: UIA2 Integrity function, part 1}
\end{figure}

\begin{figure}
\includegraphics[alt={},max width=\textwidth]{https://cdn.mathpix.com/cropped/385f8b5b-fd9b-4f2d-b7fd-b443bf1750cf-22.jpg?height=1395&width=1102&top_left_y=335&top_left_x=504}
\captionsetup{labelformat=empty}
\caption{Figure 3: UIA2 Integrity function, part 2}
\end{figure}

\section*{ANNEX 4 \\ Simulation Program Listing}

\subsection*{4.1. UEAII}

\subsection*{4.1.1 Header File}
```
/*-------------------------------------------------------
    * f8.h
    *------------------------------------------------------*/
#ifndef F8_H_
#define F8_H_
#include "SNOW_3G.h"
/* f8.
    * Input key: 128 bit Confidentiality Key.
    * Input count:32-bit Count, Frame dependent input.
    * Input bearer: 5-bit Bearer identity (in the LSB side).
    * Input dir:1 bit, direction of transmission.
    * Input data: length number of bits, input bit stream.
    * Input length: 16 bit Length, i.e., the number of bits to be encrypted or
    * decrypted.
    * Output data: Output bit stream. Assumes data is suitably memory
    * allocated.
    * Encrypts/decrypts blocks of data between 1 and 20000 bits in length as
    * defined in Section 3.
    */
void f8( u8 *key, int count, int bearer, int dir, u8 *data, int length );
#endif
```


\subsection*{4.1.2 Code}
```
/*------------------------------------------------------
    * f8.c
    *-------------------------------------------------------*/
#include "f8.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
/* f8.
    * Input key: 128 bit Confidentiality Key.
    * Input count:32-bit Count, Frame dependent input.
    * Input bearer: 5-bit Bearer identity (in the LSB side).
    * Input dir:1 bit, direction of transmission.
    * Input data: length number of bits, input bit stream.
    * Input length: 16 bit Length, i.e., the number of bits to be encrypted or
    * decrypted.
    * Output data: Output bit stream. Assumes data is suitably memory
    * allocated.
    * Encrypts/decrypts blocks of data between 1 and 20000 bits in length as
    * defined in Section 3.
    */
void f8( u8 *key, int count, int bearer, int dir, u8 *data, int length )
{
    u32 K[4],IV[4];
        int n = ( length + 31 ) / 32;
        int i=0;
        u32 *KS;
```

```
    /*Initialisation*/
    /* Load the confidentiality key for SNOW 3G initialization as in section
3.4. */
    memcpy(K+3,key+0,4); /*K[3] = key[0]; we assume
K[3]=key[0]||key[1]||...||key[31] , with key[0] the
            * most important bit of key*/
    memcpy(K+2,key+4,4); /*K[2] = key[1];*/
    memcpy(K+1, key+8,4); /*K[1] = key[2];*/
    memcpy(K+0, key+12,4); /*K[0] = key[3]; we assume
K[0]=key[96]||key[97]||...||key[127], with key[127] the
            * least important bit of key*/
    /* Prepare the initialization vector (IV) for SNOW 3G initialization as in
section 3.4. */
    IV[3] = count;
    IV[2] = (bearer << 27) | ((dir & 0x1) << 26);
    IV[1] = IV[3];
    IV[0] = IV[2];
    /* Run SNOW 3G algorithm to generate sequence of key stream bits KS*/
    Initialize(K, IV);
    KS = (u32 *)malloc(4*n);
    GenerateKeystream(n, (u32*) KS);
    /* Exclusive-OR the input data with keystream to generate the output bit
stream */
    for (i=0;i<n*4;i++)
        data[i] ^= *(((u8*)KS)+i);
    free(KS);
}
/* End of f8.c */
```


\subsection*{4.2. UIAII}

\subsection*{4.2.1 Header File}
```
/*-------------------------------------------------------
    * f9.h
    *------------------------------------------------------*/
#ifndef F9_H_
#define F9_H_
#include "SNOW_3G.h"
/* f9.
    * Input key: 128 bit Integrity Key.
    * Input count:32-bit Count, Frame dependent input.
    * Input fresh: 32-bit Random number.
    * Input dir:1 bit, direction of transmission (in the LSB).
    * Input data: length number of bits, input bit stream.
    * Input length: 64 bit Length, i.e., the number of bits to be MAC'd.
    * Output : 32 bit block used as MAC
    * Generates 32-bit MAC using UIA2 algorithm as defined in Section 4.
    */
u8* f9( u8* key, int count, int fresh, int dir, u8 *data, u64 length);
#endif
```


\subsection*{4.2.2 Code}
```
/*------------------------------------------------------
    * f9.C
    *--------------------------------------------------------*/
#include "f9.h"
#include <stdio.h>
#include <math.h>
#include <string.h>
/* MUL64x.
    * Input V: a 64-bit input.
    * Input c: a 64-bit input.
    * Output : a 64-bit output.
    * A 64-bit memory is allocated which is to be freed by the calling
    * function.
    * See section 4.3.2 for details.
    */
u64 MUL64x(u64 V, u64 c)
{
        if ( V & 0x8000000000000000 )
            return (V << 1) ^ c;
        else
            return V << 1;
}
/* MUL64xPOW.
    * Input V: a 64-bit input.
    * Input i: a positive integer.
    * Input c: a 64-bit input.
    * Output : a 64-bit output.
    * A 64-bit memory is allocated which is to be freed by the calling
function.
    * See section 4.3.3 for details.
    */
u64 MUL64xPOW(u64 V, u8 i, u64 c)
{
        if ( i == 0)
                return V;
        else
                return MUL64x( MUL64xPOW(V,i-1,c) , c);
}
/* MUL64.
    * Input V: a 64-bit input.
    * Input P: a 64-bit input.
    * Input c: a 64-bit input.
    * Output : a 64-bit output.
    * A 64-bit memory is allocated which is to be freed by the calling
    * function.
    * See section 4.3.4 for details.
    */
u64 MUL64(u64 V, u64 P, u64 c)
{
        u64 result = 0;
        int i = 0;
        for ( i=0; i<64; i++)
            {
                if( ( P>>i ) & 0x1 )
                    result ^= MUL64xPOW(V,i,c);
        }
```

```
        return result;
}
/* mask32bit.
    * Input n: an integer in 1-32.
    * Output : a 32 bit mask.
    * Prepares a 32 bit mask with required number of 1 bits on the MSB side.
    */
u32 mask32bit(int n)
{
    u32 mask=0x0;
    if ( n%32 == 0 )
        return 0xffffffff;
    while (n--)
            mask = (mask>>1) ^ 0x80000000;
    return mask;
}
/* f9.
    * Input key: 128 bit Integrity Key.
    * Input count:32-bit Count, Frame dependent input.
    * Input fresh: 32-bit Random number.
    * Input dir:1 bit, direction of transmission (in the LSB).
    * Input data: length number of bits, input bit stream.
    * Input length: 64 bit Length, i.e., the number of bits to be MAC'd.
    * Output : 32 bit block used as MAC
    * Generates 32-bit MAC using UIA2 algorithm as defined in Section 4.
    */
u8* f9( u8* key, int count, int fresh, int dir, u8 *data, u64 length)
{
        u32 K[4],IV[4], z[5];
        int i=0,D;
        static u32 MAC_I = 0; /* static memory for the result */
        u64 EVAL;
        u64 V;
        u64 P;
        u64 Q;
        u64 c;
        u64 M_D_2;
        int rem_bits = 0;
        u32 mask = 0;
        u32 *message;
    message = (u32*)data; /* To operate 32 bit message internally. */
        /* Load the Integrity Key for SNOW3G initialization as in section 4.4. */
        memcpy(K+3, key+0,4); /*K[3] = key[0]; we assume
K[3]=key[0]||key[1]||...||key[31], with key[0] the
                    * most important bit of key*/
        memcpy(K+2,key+4,4); /*K[2] = key[1];*/
    memcpy(K+1, key+8,4); /*K[1] = key[2];*/
    memcpy(K+0, key+12,4); /*K[0] = key[3]; we assume
K[0]=key[96]||key[97]||...||key[127], with key[127] the
                * least important bit of key*/
        /* Prepare the Initialization Vector (IV) for SNOW3G initialization as in
section 4.4. */
        IV[3] = count;
        IV[2] = fresh;
        IV[1] = count ^ ( dir << 31 ) ;
        IV[0] = fresh ^ (dir << 15);
        z[0] = z[1] = z[2] = z[3] = z[4] = 0;
        /* Run SNOW 3G to produce 5 keystream words z_1, z_2, z_3, z_4 and z_5. */
```

```
UEA2 and UIA2 Specification Version 1.1
```

```
    Initialize(K,IV);
    GenerateKeystream(5,z);
    P = (u64)z[0] << 32 | (u64)z[1];
    Q = (u64)z[2] << 32 | (u64)z[3];
    /* Calculation */
    D = ceil( length / 64.0 ) + 1;
    EVAL = 0;
    c = 0x1b;
    /* for 0 <= i <= D-3 */
    for (i=0;i<D-2;i++)
    {
        V = EVAL ^ ( (u64)message[2*i] << 32 | (u64)message[2*i+1] );
        EVAL = MUL64(V,P,C);
    }
    /* for D-2 */
    rem_bits = length % 64;
    if (rem_bits == 0)
        rem_bits = 64;
    mask = mask32bit(rem_bits%32);
    if (rem_bits > 32)
    {
        M_D_2 = ( (u64) message[2*(D-2)] << 32 ) | (u64) (message[2*(D-2)+1] &
mask) ;
    } else
    {
        M_D_2 = ( (u64) message[2*(D-2)] & mask) << 32 ;
    }
    V = EVAL ^ M_D_2;
    EVAL = MUL64(V,P,C);
    /* for D-1 */
    EVAL ^= length;
    /* Multiply by Q */
    EVAL = MUL64(EVAL,Q,C);
    MAC_I = (u32) (EVAL >> 32) ^ z[4];
    return (u8*) &MAC_I;
}
/* End of f9.c */
/*--------------------------------------------------------------------------
```
