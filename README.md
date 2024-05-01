# Mission Statement
Make using Bouncy Castle with OpenPGP great fun again!

This project gives you the following super-powers

encrypt, decrypt, sign and verify GPG/PGP files with just a few lines of code protect all the data at rest by reading encrypted files with transparent GPG decryption you can even decrypt a gpg encrypted ZIP and re-encrypt each file in it again -- never again let plaintext hit your servers disk!

# Examples
Bouncy GPG comes with several examples build in.

## Key management
Bouncy GPG supports reading gpg keyrings and parsing keys exported via gpg --export and gpg --export-secret-key.

The unit tests have some examples creating/reading keyrings.

The easiest way to manage keyrings is to use the pre-defined KeyringConfigs.

## Encrypt & sign a file and then decrypt it & validate the signature
The following snippet encrypts a secret message to recipient@example.com (and also self-encrypts it to sender@example.com), and signs with sender@example.com.

The encrypted message is then decrypted and the signature is verified. (This is from a documentation test).

final String original_message = "I love deadlines. I like the whooshing sound they make as they fly by. Douglas Adams";

// Most likely you will use one of the KeyringConfigs.... methods. // These are wrappers for the test. KeyringConfig keyringConfigOfSender = Configs .keyringConfigFromResourceForSender();

ByteArrayOutputStream result = new ByteArrayOutputStream();

try ( BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(result, 16384 * 1024);

final OutputStream outputStream = BouncyGPG .encryptToStream() .withConfig(keyringConfigOfSender) .withStrongAlgorithms() .toRecipients("recipient@example.com", "sender@example.com") .andSignWith("sender@example.com") .binaryOutput() .andWriteTo(bufferedOutputStream); // Maybe read a file or a webservice? final ByteArrayInputStream is = new ByteArrayInputStream(original_message.getBytes()) ) { Streams.pipeAll(is, outputStream); // It is very important that outputStream is closed before the result stream is read. // The reason is that GPG writes the signature at the end of the stream. // This is triggered by closing the stream. // In this example outputStream is closed via the try-with-resources mechanism of Java }

result.close(); byte[] chipertext = result.toByteArray();

//////// Now decrypt the stream and check the signature

// Most likely you will use one of the KeyringConfigs.... methods. // These are wrappers for the test. KeyringConfig keyringConfigOfRecipient = Configs .keyringConfigFromResourceForRecipient();

final OutputStream output = new ByteArrayOutputStream(); try ( final InputStream cipherTextStream = new ByteArrayInputStream(chipertext);

final BufferedOutputStream bufferedOut = new BufferedOutputStream(output);

final InputStream plaintextStream = BouncyGPG .decryptAndVerifyStream() .withConfig(keyringConfigOfRecipient) .andRequireSignatureFromAllKeys("sender@example.com") .fromEncryptedInputStream(cipherTextStream) ) { Streams.pipeAll(plaintextStream, bufferedOut); }

output.close(); final String decrypted_message = new String(((ByteArrayOutputStream) output).toByteArray());

assertEquals(original_message, decrypted_message);

## Performance
Bouncy castle is often fast enough to not be the bottleneck. That said, here are some metrics to give you an indication of the performance:

Use Case MBP 2,9 GHz Intel Core i5, Java 1.8.0_111 (please add more via PR) Encrypt & sign 1GB random ~64s (16 MB/s) Decrypt 1GB random ~32s (32 MB/s)

# Demos
The directory examples contains several examples that show how easy some common use cases are implemented.

## demo_decrypt.sh
Decrypt a file and verify the signature.

decrypt.sh SOURCEFILE DESTFILE Uses the testing keys to decrypt a file. Useful for performance measurements and gpg interoperability.

## demo_encrypt.sh
Encrypt and sign a file.

encrypt.sh SOURCEFILE DESTFILE Uses the testing keys to encrypt a file. Useful for performance measurements and gpg interoperability.

## demo_reencrypt.sh
A GPG encrypted ZIP file is decrypted on the fly. The structure of the ZIP is then written to disk. All files are re-encrypted before saving them.

demo_reencrypt.sh TARGET -- decrypts an encrypted ZIP file containing three files (total size: 1.2 GB) AND re-encrypts each of the files in the ZIP to the TARGET dir. The sample shows how e.g. batch jobs can work with large files without leaving plaintext on disk (together with Transparent GPG decryption).

This scheme has some very appealing benefits:

Data in transit is always encrypted with public key cryptography. Indispensable when you have to use ftp, comforting when you use https and the next Heartbleed pops up. Data at rest is always encrypted with public key cryptography. When (not if) you get hacked, this can make all the difference between "Move along folks, nothing to see here!" and "I lost confidential customer data to the competition". You still need to protect the private keys, but this is considerable easier than the alternatives. Consider the following batch job:

The customer sends a large (several GB) GPG encrypted ZIP archive containing a directory structure with several data files Your pre-processing needs to split up the data for further processing pre-processing stream-processes the GPG/ZIP archive The GPG stream is decrypted using the BouncyGPG.decryptAndVerifyStream() InputStream The ZIP file is processed with ExplodeAndReencrypt Each file from the archive is processed And transparently encrypted with GPG and stored for further processing The processing job transparently reads the files without writing plaintext to the disk.

# HOWTO
Have a look at the example classes to see how easy it is to use Bouncy Castle PGP.

## #1 Register Bouncy Castle Provider
Add bouncy castle as a dependency and then install the provider before in your application.

Add Build Dependency Gradle // build.gradle // in build.gradle add a dependency to bouncy castle and bouncy-gpg

//...

repositories { mavenCentral() jcenter() }

//...

// ... dependencies { compile 'org.bouncycastle:bcprov-jdk15on:1.59' compile 'org.bouncycastle:bcpg-jdk15on:1.59' // ... compile 'name.neuhalfen.projects.crypto.bouncycastle.openpgp:bouncy-gpg:2.+' // ... } Maven Dropping this in the root level of pom.xml lets you use this lib in a maven project:

bintray bintray false http://jcenter.bintray.com and this dependency snippet: name.neuhalfen.projects.crypto.bouncycastle.openpgp bouncy-gpg 2.1.0 Install Provider // in one of you classed install the BC provider if (Security.getProvider(BouncyCastleProvider.PROVIDER_NAME) == null) { Security.addProvider(new BouncyCastleProvider()); } #2 Important Classes Class Use when you want to BouncyGPG Starting point for the convenient fluent en- and decryption API. KeyringConfigs Create default implementations for GPG keyring access. You can also create your own implementations by implementing KeyringConfig. KeyringConfigCallbacks Used by KeyringConfigs. Create default implementations to provide secret-key passwords. DefaultPGPAlgorithmSuites Select from predefined algorithms suites or create your won with PGPAlgorithmSuite. ReencryptExplodedZipSinglethread Work with encrypted ZIPs FAQ Why should I use this? For common use cases this project is easier than vanilla Bouncy Castle. It also has a pretty decent unit test coverage. It is free (speech & beer). Can I just grab a class or two for my project? Sure! Just grab it and hack away! The code is placed under the Apache License 2.0, you can't get much more permissive than this. Why is the test coverage so low? Test coverage for 'non-example' code is >85%. Most of the not tested cases are either trivial OR lines that throw exceptions when the input format is broken. How can I contribute? Pullrequests are welcome! Please state in your PR that you put your code under the LICENSE. I am getting 'org.bouncycastle.openpgp.PGPException: checksum mismatch ..' exceptions The passphrase to your private key is very likely wrong (or you did not pass a passphrase). I am getting 'java.security.InvalidKeyException: Illegal key size' / 'java.lang.SecurityException: Unsupported keysize or algorithm parameters' The unrestricted policy files for the JVM are probably not installed. I am getting 'java.io.EOFException: premature end of stream in PartialInputStream' while decrypting / Sender can't validate signature This often happens when encrypting to a 'ByteArrayOutputStream' and the encryption stream is not propely closed. The reason is that GPG writes the signature at the end of the stream. This is triggered by closing the stream. Where is 'secring.pgp'? 'secring.gpg' has been removed in gpg 2.1. Use the other methods to read private keys. Building The project is a basic gradle build. All the scripts use ./gradlew installDist

The coverage report (incl. running tests) is generated with ./gradlew check.

Publish to jcenter ./gradlew bintrayUpload

CAVE Only one keyring per userid ("sender@example.com") supported. Only one signing key per userid supported. TODOs LICENSE This code is placed under the Apache License 2.0. Don't forget to adhere to the BouncyCastle License (http://bouncycastle.org/license.html).
