## Testing Data Storage on Android

The protection of sensitive data, such as authentication tokens or private information, is a key focus in mobile security. In this chapter you will learn about the APIs Android offers for local data storage, as well as best practices for using them.

Note that "sensitive data" need to be identified in the context of each specific app. Data classification is described in detail in the chapter "Testing Processes and Techniques".

### Testing for Sensitive Data in Local Storage

#### Overview

Conventional wisdom suggests saving as little sensitive data as possible on permanent local storage. However, in most practical scenarios, at least some types of user-related data need to be stored. For example, asking the user to enter a highly complex password every time the app is started isn't a great idea from a usability perspective. As a result, most apps must locally cache some kind of authentication token. Other types of sensitive data, such as personally identifiable information (PII), might also be saved if the particular scenario calls for it.

A vulnerability occurs when sensitive data is not properly protected by an app when persistently storing it. The app might be able to store it in different places, for example locally on the device or on an external SD card. When trying to exploit these kinds of issues, consider that there might be a lot of information processed and stored in different locations. It is important to identify at the beginning what kind of information is processed by the mobile application and keyed in by the user and what might be interesting and valuable for an attacker (e.g. passwords, credit card information, PII).

Consequences for disclosing sensitive information can be various, like disclosure of encryption keys that can be used by an attacker to decrypt information. More generally speaking an attacker might be able to identify this information to use it as a basis for further attacks like social engineering (when PII is disclosed), account hijacking (if session information or an authentication token is disclosed) or gather information from apps that have a payment option in order to attack and abuse it.

[Storing data](https://developer.android.com/guide/topics/data/data-storage.html "Storing Data in Android") is essential for many mobile apps, for example in order to keep track of user settings or data a user has keyed in that needs to be stored locally or offline. Data can be stored persistently in various ways. The following list shows those mechanisms that are widely used on the Android platform:

- Shared Preferences
- Internal Storage
- External Storage
- SQLite Databases
- Realm Databases

The following snippets of code demonstrate bad practices that disclose sensitive information, but also show the different storage mechanisms on Android in detail.

##### Shared Preferences

[Shared Preferences](http://developer.android.com/reference/android/content/SharedPreferences.html "Shared Preferences") is a common approach to store Key/Value pairs persistently in the filesystem by using an XML structure. Within an activity the following code might be used to store sensitive information like a username and a password:

```java
SharedPreferences sharedPref = getSharedPreferences("key", MODE_WORLD_READABLE);
SharedPreferences.Editor editor = sharedPref.edit();
editor.putString("username", "administrator");
editor.putString("password", "supersecret");
editor.commit();
```

Once the activity is called, the file key.xml is created with the provided data. This code is violating several best practices.

- The username and password is stored in clear text in `/data/data/<package-name>/shared_prefs/key.xml`

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
  <string name="username">administrator</string>
  <string name="password">supersecret</string>
</map>
```

- `MODE_WORLD_READABLE` allows all applications to access and read the content of `key.xml`

```bash
root@hermes:/data/data/sg.vp.owasp_mobile.myfirstapp/shared_prefs # ls -la
-rw-rw-r-- u0_a118 u0_a118    170 2016-04-23 16:51 key.xml
```

> Please note that `MODE_WORLD_READABLE` and `MODE_WORLD_WRITEABLE` were deprecated in API 17. Although this may not affect newer devices, applications compiled with android:targetSdkVersion set prior to 17 may still be affected, if they run on OS prior to Android 4.2 (`JELLY_BEAN_MR1`).


##### SQLite Database (Unencrypted)

SQLite is a SQL database that stores data to a `.db` file. The Android SDK comes with built in support for SQLite databases. The main package to manage the databases is `android.database.sqlite`.
Within an activity the following code might be used to store sensitive information like a username and a password:

```java
SQLiteDatabase notSoSecure = openOrCreateDatabase("privateNotSoSecure",MODE_PRIVATE,null);
notSoSecure.execSQL("CREATE TABLE IF NOT EXISTS Accounts(Username VARCHAR, Password VARCHAR);");
notSoSecure.execSQL("INSERT INTO Accounts VALUES('admin','AdminPass');");
notSoSecure.close();
```

Once the activity is called, the database file `privateNotSoSecure` is created with the provided data and is stored in clear text in `/data/data/<package-name>/databases/privateNotSoSecure`.

There might be several files available in the databases directory, besides the SQLite database.

- [Journal files](https://www.sqlite.org/tempfiles.html "SQLite Journal files"): These are temporary files used to implement atomic commit and rollback capabilities in SQLite.
- [Lock files](https://www.sqlite.org/lockingv3.html "SQLite Lock Files"): The lock files are part of the locking and journaling mechanism designed to improve concurrency in SQLite and to reduce the writer starvation problem.

Unencrypted SQLite databases should not be used to store sensitive information.

##### SQLite Databases (Encrypted)

By using the library [SQLCipher](https://www.zetetic.net/sqlcipher/sqlcipher-for-android/ "SQLCipher") SQLite databases can be encrypted, by providing a password.

```java
SQLiteDatabase secureDB = SQLiteDatabase.openOrCreateDatabase(database, "password123", null);
secureDB.execSQL("CREATE TABLE IF NOT EXISTS Accounts(Username VARCHAR,Password VARCHAR);");
secureDB.execSQL("INSERT INTO Accounts VALUES('admin','AdminPassEnc');");
secureDB.close();

```

If encrypted SQLite databases are used, check if the password is hardcoded in the source, stored in shared preferences or hidden somewhere else in the code or file system.
A secure approach to retrieve the key, instead of storing it locally could be to either:

- Ask the user every time for a PIN or password to decrypt the database, once the app is opened (weak password or PIN is prone to brute force attacks), or
- Store the key on the server and make it accessible via a web service (then the app can only be used when the device is online).

##### Realm Databases

The [Realm Database for Java](https://realm.io/docs/java/latest/ "Realm Database") is getting more and more popular amongst developers. The database and its content can be encrypted by providing a key in the  configuration.

```java
//the getKey() method either gets the key from the server or from a Keystore, or is deferred from a password.
RealmConfiguration config = new RealmConfiguration.Builder()
  .encryptionKey(getKey())
  .build();

Realm realm = Realm.getInstance(config);

```

If encryption is not used, you should be able to obtain the data. If encryption is enabled, check if the key is hardcoded in the source or resources, whether it is stored unprotected in shared preferences or somewhere else.

##### Internal Storage

Files can be saved directly on the [internal storage](http://developer.android.com/guide/topics/data/data-storage.html#filesInternal "Using Internal Storage") of the device. By default, files saved to the internal storage are private to your app and other apps cannot access them. When the user uninstalls your app, these files are removed.
Within an activity the following code might be used to store sensitive information in the variable `test` persistently to the internal storage:

```java
FileOutputStream fos = null;
try {
   fos = openFileOutput(FILENAME, Context.MODE_PRIVATE);
   fos.write(test.getBytes());
   fos.close();
} catch (FileNotFoundException e) {
   e.printStackTrace();
} catch (IOException e) {
   e.printStackTrace();
}
```

The file mode needs to be checked to make sure that only the app itself has access to the file. This can be set by using `MODE_PRIVATE`. Other modes like `MODE_WORLD_READABLE` (deprecated) and  `MODE_WORLD_WRITEABLE` (deprecated) are more lax and can pose a security risk.

It should also be checked what files are opened and read within the app by searching for the class `FileInputStream`. Part of the internal storage mechanisms is also the cache storage. To cache data temporarily, functions like `getCacheDir()` can be used.

##### External Storage

Every Android-compatible device supports a [shared external storage](https://developer.android.com/guide/topics/data/data-storage.html#filesExternal "Using External Storage") that you can use to save files. This can be a removable storage media (such as an SD card) or an internal (non-removable) storage.
Files saved to the external storage are world-readable and can be modified by the user when they enable USB mass storage to transfer files on a computer.
Within an activity the following code might be used to store sensitive information in the file `password.txt` persistently to the external storage:

```java
File file = new File (Environment.getExternalFilesDir(), "password.txt");
String password = "SecretPassword";
FileOutputStream fos;
    fos = new FileOutputStream(file);
    fos.write(password.getBytes());
    fos.close();
```

Once the activity is called, the file is created with the provided data and the data is stored in clear text in the external storage.

It’s also worth to know that files stored outside the application folder (`data/data/<package-name>/`) will not be deleted when the user uninstall the application.


##### KeyChain

The [KeyChain class](http://developer.android.com/reference/android/security/KeyChain.html "Android KeyChain") is used to store and retrieve *system-wide* private keys and their corresponding certificate (chain). The user will be prompted to set a lock screen pin or password to protect the credential storage if it hasn’t been set, if something gets imported into the KeyChain the first time. Please note that the KeyChain is system-wide: so every application can access the materials stored in the KeyChain.

##### KeyStore

The [Android KeyStore](http://www.androidauthority.com/use-android-keystore-store-passwords-sensitive-information-623779/ "Use Android KeyStore") provides a means of (more or less) secure credential storage. As of Android 4.3, it provides public APIs for storing and using app-private keys. An app can create a new private/public key pair to encrypt application secrets by using the public key and decrypt the same by using the private key.

The keys stored in the Android KeyStore can be protected such that the user needs to authenticate to access them. The user's lock screen credentials (pattern, PIN, password or fingerprint) are used for authentication.

Stored keys can be configured to operate in one of the two modes:

1. User authentication authorizes the use of keys for a duration of time. All keys in this mode are authorized for use as soon as the user unlocks the device.  The duration for which the authorization remains valid can be customized for each key. This option can only be used if the secure lock screen is enabled. If the user disables the secure lock screen, any stored keys become permanently invalidated.

2. User authentication authorizes a specific cryptographic operation associated with one key. In this mode, each operation involving such a key must be individually authorized by the user. Currently, the only means of such authorization is fingerprint authentication.


The level of security afforded by the Android KeyStore depends on its implementation, which differs between devices. Most modern devices offer a hardware-backed KeyStore implementation. In that case, keys are generated and used in a Trusted Execution Environment or a Secure Element and are not directly accessible for the operating system. This means that the encryption keys themselves cannot be easily retrieved even from a rooted device. You can check whether the keys are inside the secure hardware, based on the return value of the `isInsideSecureHardware()` method which is part of the [class `KeyInfo`](https://developer.android.com/reference/android/security/keystore/KeyInfo.html "Class KeyInfo"). Please note that private keys are often indeed stored correctly within the secure hardware, but secret keys, HMAC keys are, are not stored securely according to the KeyInfo on quite some devices.

In a software-only implementation, the keys are encrypted with a [per-user encryption master key](https://nelenkov.blogspot.sg/2013/08/credential-storage-enhancements-android-43.html "Nikolay Elenvok - Credential storage enhancements in Android 4.3"). In that case, an attacker can access all keys on a rooted device in the folder `/data/misc/keystore/`. As the master key is generated using the user’s own lock screen pin/password, the Android KeyStore is unavailable when the device is locked.

##### Older KeyStore Implementations

Older Android versions do not have a KeyStore, but do have the KeyStore interface from JCA (Java Cryptography Architecture). One can use various KeyStores that implement this interface and ensure secrecy and integrity to the keys stored in the KeyStore implementation. All implementations rely on the fact that a file is stored on the filesystem, which then protects its content by a password. For this, it is recommended to use the BouncyCastle KeyStore (BKS).
You can create one by using the `KeyStore.getInstance("BKS", "BC");`, where "BKS" is the KeyStore name (BouncycastleKeyStore) and "BC" is the provider (BouncyCastle). Alternatively you can use SpongyCastle as a wrapper and initialize the KeyStore: `KeyStore.getInstance("BKS", "SC");`.

Please be aware that not all KeyStores offer proper protection to the keys stored in the KeyStore files.


#### Static Analysis

##### Local Storage

As already pointed out, there are several ways to store information within Android. Several checks should therefore be applied to the source code to identify the storage mechanisms used within the Android app and whether sensitive data is processed insecurely.

- Check `AndroidManifest.xml` for permissions to read from or write to external storage, like `uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"`
- Check the source code for keywords and API calls that are used for storing data:
    - File permissions like:
      - `MODE_WORLD_READABLE` or `MODE_WORLD_WRITABLE`. IPC files should not be created with permissions of `MODE_WORLD_READABLE` or `MODE_WORLD_WRITABLE` unless it is required as any app would be able to read or write the file even though it may be stored in the app private data directory.
    - Classes and functions like:
      - `SharedPreferences` Class (Storage of key-value pairs)
      - `FileOutPutStream` Class (Using Internal or External Storage)
      - `getExternal*` functions (Using External Storage)
      - `getWritableDatabase` function (return a SQLiteDatabase for writing)
      - `getReadableDatabase` function (return a SQLiteDatabase for reading)
      - `getCacheDir` and `getExternalCacheDirs` function (Using cached files)

Encryption operations should rely on solid and tested functions provided by the SDK. The following describes different “bad practices” that should be checked in the source code:

- Check if simple bit operations are used, like XOR or Bit flipping to “encrypt” sensitive information that is stored locally. This should be avoided as the data can easily be recovered.
- Check if keys are created or used without taking advantage of the Android onboard features like the Android KeyStore.
- Check if keys are disclosed.

###### Typical Misuse: Hardcoded Cryptographic Keys

The use of a hardcoded or world-readable cryptographic key significantly increases the possibility that encrypted data may be recovered. Once it is obtained by an attacker, the task to decrypt the sensitive data becomes trivial, and the initial idea to protect confidentiality fails.

When using symmetric cryptography, the key needs to be stored within the device and it is just a matter of time and effort from the attacker to identify it.

Consider the following scenario: An application is reading and writing to an encrypted database but the decryption is done based on a hardcoded key:

```Java
this.db = localUserSecretStore.getWritableDatabase("SuperPassword123");
```

Since the key is the same for all app installations it is trivial to obtain it. The advantages of having sensitive data encrypted are gone, and there is effectively no benefit in using encryption in this way. Similarly, look for hardcoded API keys/private keys and other valuable pieces. Encoded/encrypted keys is just another attempt to make it harder but not impossible to get the crown jewels.

Let's consider this piece of code:

```Java
//A more complicated effort to store the XOR'ed halves of a key (instead of the key itself)
private static final String[] myCompositeKey = new String[]{
  "oNQavjbaNNSgEqoCkT9Em4imeQQ=","3o8eFOX4ri/F8fgHgiy/BS47"
};
```

Algorithm to decode the original key in this case might look like this:

```Java
public void useXorStringHiding(String myHiddenMessage) {
  byte[] xorParts0 = Base64.decode(myCompositeKey[0],0);
  byte[] xorParts1 = Base64.decode(myCompositeKey[1],0);

  byte[] xorKey = new byte[xorParts0.length];
  for(int i = 0; i < xorParts1.length; i++){
    xorKey[i] = (byte) (xorParts0[i] ^ xorParts1[i]);
  }
  HidingUtil.doHiding(myHiddenMessage.getBytes(), xorKey, false);
}
```

Verify common places where secrets are usually hidden:
- resources (typically at res/values/strings.xml)

Example:
```xml
<resources>
    <string name="app_name">SuperApp</string>
    <string name="hello_world">Hello world!</string>
    <string name="action_settings">Settings</string>
    <string name="secret_key">My_Secret_Key</string>
  </resources>
```

- shared preferences (typically in the shared_prefs directory)

- build configs, such as in local.properties or gradle.properties

Example:
```
buildTypes {
  debug {
    minifyEnabled true
    buildConfigField "String", "hiddenPassword", "\"${hiddenPassword}\""
  }
}
```

##### KeyChain and Android KeyStore

When going through the source code it should be analyzed if native mechanisms that are offered by Android are applied to the identified sensitive information. Sensitive information must not be stored in clear text but should be encrypted. If sensitive information needs to be stored on the device itself, several API calls are available to protect the data on the Android device by using the **KeyChain** and **KeyStore**. The following should therefore be done:

- Check if a key pair is created within the app by looking for the class `KeyPairGenerator`.
- Check that the app is using the Android KeyStore and Cipher mechanisms to securely store encrypted information on the device. Look for the pattern `import java.security.KeyStore`, `import javax.crypto.Cipher`, `import java.security.SecureRandom` and corresponding usages.
- The `store(OutputStream stream, char[] password)` function can be used to store the KeyStore to disk with a specified password. Check that the password provided is not hardcoded and is defined by user input as this should only be known to the user. Look for the function `.store()`.

#### Dynamic Analysis

Install and use the app as it is intended and execute all functions at least once. Data can be generated when entered by the user, sent by the endpoint or it is already shipped within the app when installing it. Afterwards check the following items:

- Check the files that are shipped with the mobile application once installed in `/data/data/<package-name>/` in order to identify development, backup or simply old files that shouldn’t be in a production release.
- Check if SQLite databases are available and if they contain sensitive information (usernames, passwords, keys etc.). SQLite databases are stored in `/data/data/<package-name>/databases`.
- Check Shared Preferences that are stored as XML files in the shared_prefs directory of the app for sensitive information, which is in `/data/data/<package-name>/shared_prefs`.
- Check the file system permissions of the files in `/data/data/<package-name>`. Only the user and group created when installing the app (e.g. u0_a82) should have the user rights read, write and execute (`rwx`). Others should have no permissions to files, but may have the executable flag to directories.
- Check if there is a Realm database available in `/data/data/<package-name>/files/`and if it is unencrypted and contains sensitive information. The file extension is `realm` and the file name is `default` by default. Inspecting the Realm database is done with the [Realm Browser](https://github.com/realm/realm-browser-osx "Realm Browser for macOS").

#### Remediation

The credo for saving data can be summarized quite easily: Public data should be available for everybody, but sensitive and private data needs to be protected or even better not get stored on the device in the first place.

If sensitive information (credentials, keys, PII, etc.) is needed locally on the device several best practices are offered by Android that should be used to store data securely instead of reinventing the wheel or leaving data unencrypted on the device.

The following is a list of best practices used for secure storage of certificates, keys and sensitive data in general:

- Encryption or decryption functions that were self implemented need to be avoided. Instead use Android implementations such as [Cipher](https://developer.android.com/reference/javax/crypto/Cipher.html "Cipher Class"), [SecureRandom](https://developer.android.com/reference/java/security/SecureRandom.html "SecureRandom Class") and [KeyGenerator](https://developer.android.com/reference/javax/crypto/KeyGenerator.html "KeyGenerator Class").   
- Username and password should not be stored on the device. Instead, perform initial authentication using the username and password supplied by the user, and then use a short-lived, service-specific authorization token or session identifier. If possible, use the [AccountManager](https://developer.android.com/reference/android/accounts/AccountManager.html "AccountManager Class") class to invoke a cloud-based service and do not store passwords on the device.
- Usage of `MODE_WORLD_WRITEABLE` or `MODE_WORLD_READABLE` should generally be avoided for files. If data needs to be shared with other applications, a content provider should be considered. A content provider offers read and write permissions to other apps and can make dynamic permission grants on a case-by-case basis.
- The usage of Shared Preferences or other mechanisms that are not able to protect data should be avoided to store sensitive information. SharedPreferences are insecure and not encrypted by default. [Secure-preferences](https://github.com/scottyab/secure-preferences "Secure-preferences encrypts the values of Shared Preferences") can be used to encrypt the values stored within Shared Preferences, but the Android KeyStore should be the first option to store data securely.
- Do not use the external storage for sensitive data. By default, files saved to the internal storage are private to your application and other applications cannot access them (nor can the user). When the user uninstalls your application, these files are also removed.
- To provide additional protection for sensitive data, you might choose to encrypt local files using a key that is not directly accessible to the application. For example, a key can be placed in a KeyStore and protected with a user password that is not stored on the device. While this does not protect data from a root compromise that can monitor the user keying in the password, it can provide protection for a lost device without file system encryption.
- Set variables that use sensitive information to null once finished.
- Use immutable objects for sensitive data so it cannot be changed.

Please also check the [Security Tips data](http://developer.android.com/training/articles/security-tips.html#StoringData "Security Tips for Storing Data") in the Android developers guide.


#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M1-Improper_Platform_Usage
- M2 - Insecure Data Storage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M2-Insecure_Data_Storage

##### OWASP MASVS
- V2.1: "System credential storage facilities are used appropriately to store sensitive data, such as user credentials or cryptographic keys."

##### CWE
- CWE-311 - Missing Encryption of Sensitive Data
- CWE-312 - Cleartext Storage of Sensitive Information
- CWE-522 - Insufficiently Protected Credentials
- CWE-922 - Insecure Storage of Sensitive Information

##### Tools
- Sqlite3 - http://www.sqlite.org/cli.html
- Realm Browser - Realm Browser - https://github.com/realm/realm-browser-osx


### Testing for Sensitive Data in Logs

#### Overview

There are many legit reasons to create log files on a mobile device, for example to keep track of crashes or errors or simply for usage statistics. Log files can be stored locally when being offline and being sent to the endpoint once being online again. However, logging sensitive data such as usernames or session IDs might expose the data to attackers or malicious applications and violates the confidentiality of the data.
Log files can be created in various ways and the following list shows the mechanisms that are available on Android:

- [Log Class](https://developer.android.com/reference/android/util/Log.html "Log Class")
- [Logger Class](https://developer.android.com/reference/java/util/logging/Logger.html "Logger Class")
- System.out/System.err.print

#### Static Analysis

The source code should be checked for logging mechanisms used within the Android App, by searching for the following keywords:

1. Functions and classes like:
  * `android.util.Log`
  * `Log.d` | `Log.e` | `Log.i` | `Log.v` | `Log.w` | `Log.wtf`
  * `Logger`
  * `System.out.print` | `System.err.print`

2. Keywords and system output to identify non-standard log mechanisms like:
  * logfile
  * logging
  * logs

#### Dynamic Analysis

Use the mobile app extensively so that all functionality is at least triggered once. Afterwards identify the data directory of the application in order to look for log files (`/data/data/<package-name>`). Check if log data is generated by checking the application logs, as some mobile applications create and store their own logs in the data directory.  

Many application developers still use `System.out.println()` or `printStackTrace()` instead of a proper logging class. Therefore, the testing approach also needs to cover all output generated by the application during starting, running and closing of it. In order to verify what data is printed directly by using `System.out.println()` or `printStackTrace()` the tool [`Logcat`](http://developer.android.com/tools/debugging/debugging-log.html "Debugging with Logcat") can be used to check the app output. Two different approaches are available to execute Logcat.
  * Logcat is already part of _Dalvik Debug Monitor Server_ (DDMS) and is built into Android Studio. If the app is in debug mode and running, the log output is shown in the Android Monitor in the Logcat tab. Patterns can be defined in Logcat to filter the log output of the app.

![Log output in Android Studio](Images/Chapters/0x05d/log_output_Android_Studio.png)

  * Logcat can be executed by using adb in order to store the log output permanently.

```bash
$ adb logcat > logcat.log
```

#### Remediation

Ensure that a centralized logging class and mechanism is used and that logging statements are removed from the production release, as logs may be interrogated or readable by other applications. Tools like `ProGuard`, which is already included in Android Studio can be used to strip out logging portions in the code when preparing the production release. For example, to remove logging calls implemented with the class `android.util.Log`, simply add the following option in the _proguard-project.txt_ configuration file of ProGuard:

```java
-assumenosideeffects class android.util.Log
{
public static boolean isLoggable(java.lang.String, int);
public static int v(...);
public static int i(...);
public static int w(...);
public static int d(...);
public static int e(...);
public static int wtf(...);
}
```

Please note that the above example only ensures that calls to the methods offered by the Log class will be removed. However, if the string to be logged is dynamically constructed, the code for constructing the string might remain in the bytecode. For example, the following code issues an implicit StringBuilder to construct the log statement:

```java
Log.v("Private key [byte format]: " + key);
```

The compiled bytecode however, is equivalent to the bytecode of the following log statement, which has the string constructed explicitly:

```java
Log.v(new StringBuilder("Private key [byte format]: ").append(key.toString()).toString());
```

What ProGuard guarantees is the removal of the ```Log.v``` method call. Whether the rest of the code (```new StringBuilder ...```) will be removed depends on the complexity of the code and the [ProGuard version used](https://stackoverflow.com/questions/6009078/removing-unused-strings-during-proguard-optimisation "Removing unused strings during ProGuard optimization ").

This is potentially a security risk, as the (now unused) string leaks plain text data in memory which can be accessed over a debugger or by memory dumping.

Unfortunately, there is no silver bullet against this issue, but there are few options available:

- Implement a custom logging facility that takes simple arguments and does the construction of the log statements internally.
```java
SecureLog.v("Private key [byte format]: ", key);
```
Then configure ProGuard to strip its calls.

- Remove logs on source level, instead of compiled bytecode level. Below is a simple Gradle task which comments out all log statements including any inline string builder.

```
afterEvaluate {
  project.getTasks().findAll { task -> task.name.contains("compile") && task.name.contains("Release")}.each { task ->
    task.dependsOn('removeLogs')
  }

  task removeLogs() {
    doLast {
      fileTree(dir: project.file('src')).each { File file ->
        def out = file.getText("UTF-8").replaceAll("((android\\.util\\.)*Log\\.([ewidv]|wtf)\\s*\\([\\S\\s]*?\\)\\s*;)", "/*\$1*/")
        file.write(out);
      }
    }
  }
```

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M1-Improper_Platform_Usage
- M2 - Insecure Data Storage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M2-Insecure_Data_Storage

##### OWASP MASVS
- V2.2: "No sensitive data is written to application logs."

##### CWE
- CWE-117: Improper Output Neutralization for Logs
- CWE-532: Information Exposure Through Log Files
- CWE-534: Information Exposure Through Debug Log Files

##### Tools
- ProGuard - http://proguard.sourceforge.net/
- Logcat - http://developer.android.com/tools/help/logcat.html


### Testing Whether Sensitive Data is Sent to Third Parties

#### Overview

Different 3rd party services are available that can be embedded into the app to implement different features. These features can vary from tracker services to monitor the user behavior within the app, selling banner advertisements or to create a better user experience. Interacting with these services abstracts the complexity and neediness to implement the functionality on its own and to reinvent the wheel.

The downside is that a developer doesn’t know in detail what code is executed via 3rd party libraries and therefore giving up visibility. Consequently it should be ensured that not more information as needed is sent to the service and that no sensitive information is disclosed.

3rd party services are mostly implemented in two ways:
- By using a standalone library, like a Jar in an Android project that is getting included into the APK.
- By using a full SDK.

#### Static Analysis

Some 3rd party libraries can be automatically integrated into the app through a wizard within the IDE. The permissions set in the `AndroidManifest.xml`  when installing a library through an IDE wizard should be reviewed. Especially permissions to access `SMS (READ_SMS)`, contacts (`READ_CONTACTS`) or the location (`ACCESS_FINE_LOCATION`) should be challenged if they are really needed to make the library work at a bare minimum, see also `Testing App Permissions`. When talking to developers it should be shared to them that it’s actually necessary to have a look at the differences on the project source code before and after the library was installed through the IDE and what changes have been made to the code base.

The same thing applies when adding a library or SDK manually. The source code should be checked for API calls or functions provided by the 3rd party library or SDK. The applied code changes should be reviewed and it should be checked if available security best practices of the library and SDK are applied and used.

The libraries loaded into the project should be reviewed in order to identify with the developers if they are needed and also if they are out of date and contain known vulnerabilities.

#### Dynamic Analysis

All requests made to external services should be analyzed if any sensitive information is embedded into them.
Dynamic analysis can be performed by launching a Man-in-the-middle (MITM) attack using _Burp Suite Professional_ or _OWASP ZAP_, to intercept the traffic exchanged between client and server. Once we are able to route the traffic to the interception proxy, we can try to sniff the traffic from the app to the server and vice versa. When using the app all requests that are not going directly to the server where the main function is hosted should be checked, if any sensitive information is sent to a 3rd party. This could be for example PII (Personal Identifiable Information) in a tracker or ad service.

#### Remediation

All data that is sent to 3rd Party services should be anonymized, so no PII data is available. Also all other data, like IDs in an application that can be mapped to a user account or session should not be sent to a third party.  
`AndroidManifest.xml` should only contain the permissions that are absolutely needed to work properly and as intended.

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M1-Improper_Platform_Usage
- M2 - Insecure Data Storage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M2-Insecure_Data_Storage

##### OWASP MASVS
- V2.3: "No sensitive data is shared with third parties unless it is a necessary part of the architecture."

##### CWE
- CWE-359 - Exposure of Private Information ('Privacy Violation')

##### Info
- [1] Bulletproof Android, Godfrey Nolan - Chapter 7, Third-Party Library Integration

##### Tools
- Burp Suite Professional - https://portswigger.net/burp/
- OWASP ZAP - https://www.owasp.org/index.php/OWASP_Zed_Attack_Proxy_Project


### Testing Whether the Keyboard Cache Is Disabled for Text Input Fields

#### Overview

When typing in data into input fields, the software keyboard automatically suggests what data the user might want to key in. This feature can be very useful in messaging apps to write text messages more efficiently. For input fields that are asking for sensitive information like credit card data the keyboard cache might disclose sensitive information already when the input field is selected.

#### Static Analysis

In the layout definition of an activity, `TextViews` can be defined that have XML attributes. When the XML attribute `android:inputType` is set with the constant `textNoSuggestions` the keyboard cache is not shown if the input field is selected. Only the keyboard is shown and the user needs to type everything manually and nothing is suggested to him.

```xml
   <EditText
        android:id="@+id/KeyBoardCache"
        android:inputType="textNoSuggestions"/>
```


#### Dynamic Analysis

Start the app and click into the input fields that ask for sensitive data. If strings are suggested the keyboard cache is not disabled for this input field.

#### Remediation

All input fields that ask for sensitive information, should implement the following XML attribute to [disable the keyboard suggestions](https://developer.android.com/reference/android/text/InputType.html#TYPE_TEXT_FLAG_NO_SUGGESTIONS "Disable keyboard suggestions"):

```xml
android:inputType="textNoSuggestions"
```

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M1-Improper_Platform_Usage
- M2 - Insecure Data Storage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M2-Insecure_Data_Storage

##### OWASP MASVS
- V2.4: "The keyboard cache is disabled on text inputs that process sensitive data."

##### CWE
- CWE-524 - Information Exposure Through Caching



### Testing for Sensitive Data in the Clipboard

#### Overview

When keying in data into input fields, the [clipboard](https://developer.android.com/guide/topics/text/copy-paste.html "Copy and Paste in Android") can be used to copy data in. The clipboard is accessible systemwide and therefore shared between the apps. This feature can therefore be misused by malicious apps in order to get sensitive data stored in the clipboard.

#### Static Analysis

Input fields that are asking for sensitive information need to be identified and afterwards be investigated if any countermeasures are in place to mitigate the clipboard of showing up. See the remediation section for code snippets that could be applied.

#### Dynamic Analysis

Start the app and click into the input fields that ask for sensitive data. When it is possible to get the menu to copy/paste data the functionality is not disabled for this input field.

To extract the data stored in the clipboard, the Drozer module `post.capture.clipboard` can be used:

```
dz> run post.capture.clipboard
[*] Clipboard value: ClipData.Item { T:Secretmessage }
```

#### Remediation

A general best practice is overwriting different functions in the input field to disable the clipboard specifically for it.

```Java
EditText  etxt = (EditText) findViewById(R.id.editText1);
etxt.setCustomSelectionActionModeCallback(new Callback() {

            public boolean onPrepareActionMode(ActionMode mode, Menu menu) {
                return false;
            }

            public void onDestroyActionMode(ActionMode mode) {                  
            }

            public boolean onCreateActionMode(ActionMode mode, Menu menu) {
                return false;
            }

            public boolean onActionItemClicked(ActionMode mode, MenuItem item) {
                return false;
            }
        });
```

Also `longclickable` should be deactivated for the input field.

```xml
android:longClickable="false"
```

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage
- M2 - Insecure Data Storage

##### OWASP MASVS
- V2.5: "The clipboard is deactivated on text fields that may contain sensitive data."

##### CWE
- CWE-200 - Information Exposure

##### Tools
- Drozer - https://labs.mwrinfosecurity.com/tools/drozer/


### Testing Whether Stored Sensitive Data Is Exposed via IPC Mechanisms

#### Overview

As part of the IPC mechanisms included on Android, content providers allow an app's stored data to be accessed and modified by other apps. If not properly configured, they could lead to leakage of stored sensitive data.

#### Static Analysis

The first step is to look into the `AndroidManifest.xml` in order to detect content providers exposed by the app. Content providers can be identified through the  `<provider>` element.

Check if the provider has the export tag set to "true" (`android:exported="true"`). Even if this is not the case, remember that if it has an `<intent-filter>` defined, the export tag will be automatically set to "true".

Finally, check if it is being protected by any permission tag (`android:permission`). Permission tags allow to limit the exposure to other apps.

Inspect the source code to further understand how the content provider is meant to be used. Search for the following keywords:
- `android.content.ContentProvider`
- `android.database.Cursor`
- `android.database.sqlite`
- `.query(`
- `.update(`
- `.delete(`

When exposing a content provider it should also be checked if parametrized [query methods](https://developer.android.com/reference/android/content/ContentProvider.html#query(android.net.Uri, java.lang.String[], java.lang.String, java.lang.String[], java.lang.String) "Query method in Content Provder Class") (`query()`, `update()`, and `delete()`) are being used to prevent SQL injection. If so, check if all inputs to them are properly sanitized.

As an example of a vulnerable content provider we will use the vulnerable password manager app [Sieve](https://github.com/mwrlabs/drozer/releases/download/2.3.4/sieve.apk "Sieve - Vulnerable Password Manager").

##### Inspect the AndroidManifest
Identify all defined `<provider>` elements:

```xml
<provider android:authorities="com.mwr.example.sieve.DBContentProvider" android:exported="true" android:multiprocess="true" android:name=".DBContentProvider">
    <path-permission android:path="/Keys" android:readPermission="com.mwr.example.sieve.READ_KEYS" android:writePermission="com.mwr.example.sieve.WRITE_KEYS"/>
</provider>
<provider android:authorities="com.mwr.example.sieve.FileBackupProvider" android:exported="true" android:multiprocess="true" android:name=".FileBackupProvider"/>
```

As can be seen in the `AndroidManifest.xml` above, the application exports two content providers. Note that one path ("/Keys") is being protected by read and write permissions.

##### Inspect the source code
In the `DBContentProvider.java` file the `query` function need to be inspected to detect if any sensitive information is leaked:

```java
public Cursor query(final Uri uri, final String[] array, final String s, final String[] array2, final String s2) {
    final int match = this.sUriMatcher.match(uri);
    final SQLiteQueryBuilder sqLiteQueryBuilder = new SQLiteQueryBuilder();
    if (match >= 100 && match < 200) {
        sqLiteQueryBuilder.setTables("Passwords");
    }
    else if (match >= 200) {
        sqLiteQueryBuilder.setTables("Key");
    }
    return sqLiteQueryBuilder.query(this.pwdb.getReadableDatabase(), array, s, array2, (String)null, (String)null, s2);
}
```

Here we see that there are actually two paths, "/Keys" and "/Passwords", being the latter not protected in the manifest and therefore vulnerable.

The query statement would return all passwords when accessing an URI including this path `Passwords/`. We will address this in the dynamic analysis below and find out the exact URI required.


#### Dynamic Analysis

##### Testing Content Providers

To begin dynamic analysis of an application's content providers, you should first enumerate the attack surface. This can be achieved using the Drozer module `app.provider.info` and providing the package name of the app:

```
dz> run app.provider.info -a com.mwr.example.sieve
  Package: com.mwr.example.sieve
  Authority: com.mwr.example.sieve.DBContentProvider
  Read Permission: null
  Write Permission: null
  Content Provider: com.mwr.example.sieve.DBContentProvider
  Multiprocess Allowed: True
  Grant Uri Permissions: False
  Path Permissions:
  Path: /Keys
  Type: PATTERN_LITERAL
  Read Permission: com.mwr.example.sieve.READ_KEYS
  Write Permission: com.mwr.example.sieve.WRITE_KEYS
  Authority: com.mwr.example.sieve.FileBackupProvider
  Read Permission: null
  Write Permission: null
  Content Provider: com.mwr.example.sieve.FileBackupProvider
  Multiprocess Allowed: True
  Grant Uri Permissions: False
```

In the example, two content providers are exported, each not requiring any permission to interact with them, except for the `/Keys` path in the `DBContentProvider`. Using this information you can reconstruct part of the content URIs to access the `DBContentProvider`, because it is known that they must begin with `content://`. However, the full content provider URI is not currently known.

To identify content provider URIs within the application, Drozer's `scanner.provider.finduris` module should be used. This utilizes various techniques to guess paths and determine a list of accessible content URIs:

```
dz> run scanner.provider.finduris -a com.mwr.example.sieve
Scanning com.mwr.example.sieve...
Unable to Query content://com.mwr.example.sieve.DBContentProvider/
...
Unable to Query content://com.mwr.example.sieve.DBContentProvider/Keys
Accessible content URIs:
content://com.mwr.example.sieve.DBContentProvider/Keys/
content://com.mwr.example.sieve.DBContentProvider/Passwords
content://com.mwr.example.sieve.DBContentProvider/Passwords/
```

Now that you have a list of accessible content providers, the next step is to attempt to extract data from each provider, which can be achieved using the `app.provider.query` module:

```
dz> run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --vertical
_id: 1
service: Email
username: incognitoguy50
password: PSFjqXIMVa5NJFudgDuuLVgJYFD+8w== (Base64 - encoded)
email: incognitoguy50@gmail.com
```

In addition to querying data, Drozer can be used to update, insert and delete records from a vulnerable content provider:

- Insert record

```
dz> run app.provider.insert content://com.vulnerable.im/messages
                --string date 1331763850325
                --string type 0
                --integer _id 7
```

- Update record

```
dz> run app.provider.update content://settings/secure
                --selection "name=?"
                --selection-args assisted_gps_enabled
                --integer value 0
```

- Delete record

```
dz> run app.provider.delete content://settings/secure
                --selection "name=?"
                --selection-args my_setting
```

##### SQL Injection in Content Providers

The Android platform promotes the use of SQLite databases for storing user data. Since these databases use SQL, they can be vulnerable to SQL injection. The Drozer module `app.provider.query` can be used to test for SQL injection by manipulating the projection and selection fields that are passed to the content provider:

```
dz> run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "'"
unrecognized token: "' FROM Passwords" (code 1): , while compiling: SELECT ' FROM Passwords

dz> run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --selection "'"
unrecognized token: "')" (code 1): , while compiling: SELECT * FROM Passwords WHERE (')
```

If vulnerable to SQL Injection, the application will return a verbose error message. SQL Injection in Android can be exploited to modify or query data from the vulnerable content provider. In the following example, the Drozer module `app.provider.query` is used to list all tables in the database:

```
dz> run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "*
FROM SQLITE_MASTER WHERE type='table';--"
| type  | name             | tbl_name         | rootpage | sql              |
| table | android_metadata | android_metadata | 3        | CREATE TABLE ... |
| table | Passwords        | Passwords        | 4        | CREATE TABLE ... |
| table | Key              | Key              | 5        | CREATE TABLE ... |
```

SQL Injection can also be exploited to retrieve data from otherwise protected tables:

```
dz> run app.provider.query content://com.mwr.example.sieve.DBContentProvider/Passwords/ --projection "* FROM Key;--"
| Password | pin |
| thisismypassword | 9876 |
```

These steps can be automated by using the `scanner.provider.injection` module, which automatically finds vulnerable content providers within an app:

```
dz> run scanner.provider.injection -a com.mwr.example.sieve
Scanning com.mwr.example.sieve...
Injection in Projection:
  content://com.mwr.example.sieve.DBContentProvider/Keys/
  content://com.mwr.example.sieve.DBContentProvider/Passwords
  content://com.mwr.example.sieve.DBContentProvider/Passwords/
Injection in Selection:
  content://com.mwr.example.sieve.DBContentProvider/Keys/
  content://com.mwr.example.sieve.DBContentProvider/Passwords
  content://com.mwr.example.sieve.DBContentProvider/Passwords/
```

##### File System Based Content Providers

A content provider can provide access to the underlying file system. This allows apps to share files, where the Android sandbox would otherwise prevent it. The Drozer modules `app.provider.read` and `app.provider.download` can be used to read or download files from exported file based content providers. These content providers can be susceptible to directory traversal vulnerabilities, making it possible to read otherwise protected files within the target application's sandbox.

```
dz> run app.provider.download content://com.vulnerable.app.FileProvider/../../../../../../../../data/data/com.vulnerable.app/database.db /home/user/database.db
Written 24488 bytes
```

To automate the process of finding content providers susceptible to directory traversal, the `scanner.provider.traversal` module should be used:

```
dz> run scanner.provider.traversal -a com.mwr.example.sieve
Scanning com.mwr.example.sieve...
Vulnerable Providers:
  content://com.mwr.example.sieve.FileBackupProvider/
  content://com.mwr.example.sieve.FileBackupProvider
```

Note that `adb` can also be used to query content providers on a device:

```bash
$ adb shell content query --uri content://com.owaspomtg.vulnapp.provider.CredentialProvider/credentials
Row: 0 id=1, username=admin, password=StrongPwd
Row: 1 id=2, username=test, password=test
...
```

#### Remediation

Set `android:exported` to "false" if the content is only meant to be accessed by the app itself. If not, set it to "true" and define proper read and write permissions.

Protect the content provider with the `android:protectionLevel` attribute set to `signature` protection, if it is only intended to be accessed by your own apps (signed with the same key). On the other hand, you may also want to offer access to other apps, for that you can apply a security policy by using the `<permission>` element and set a proper `android:protectionLevel`. When using `android:permission`, other applications will need to declare a corresponding `<uses-permission>` element in their own manifest to be able to interact with your content provider.

In order to avoid SQL injection attacks, use parameterized query methods such as `query()`, `update()`, and `delete()`. Be sure to properly sanitize all inputs to these methods because if, for instance, the `selection` argument is built out of user input concatenation, it could also lead to SQL injection.

You may also want to provide more granular access to other apps by using the `android:grantUriPermissions` attribute in the manifest and limit the scope with the `<grant-uri-permission>` element.

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage
- M2 - Insecure Data Storage

##### OWASP MASVS
- V2.6: "No sensitive data is exposed via IPC mechanisms."

##### CWE
- CWE-634 - Weaknesses that Affect System Processes

##### Tools
- Drozer - https://labs.mwrinfosecurity.com/tools/drozer/


### Testing for Sensitive Data Disclosure Through the User Interface

#### Overview

In many apps users need to key in different kind of data to for example register an account or execute payment. Sensitive data could be exposed if the app is not masking it properly and showing data in clear text.

Masking of sensitive data within an activity of an app should be enforced to prevent disclosure and mitigate for example shoulder surfing.

#### Static Analysis

To verify if the application is masking sensitive information that is keyed in by the user, check for the following attribute in the definition of EditText:

```
android:inputType="textPassword"
```

This will show dots in the text field instead of the keyed in characters.

#### Dynamic Analysis

To analyze if the application leaks any sensitive information to the user interface, run the application and identify parts of the app that either shows or asks for such information to be keyed in.

If the information is masked, e.g. by replacing characters in the text field through asterisks or dots the app is not leaking data to the user interface.

#### Remediation

In order to prevent leaking of passwords or pins, sensitive information should be masked in the user interface. The attribute `android:inputType="textPassword"` should therefore be used for EditText fields.

#### References

##### OWASP Mobile Top 10 2016
- M4 - Unintended Data Leakage

##### OWASP MASVS
- V2.7: "No sensitive data, such as passwords and pins, is exposed through the user interface."

##### CWE
- CWE-200 - Information Exposure


### Testing for Sensitive Data in Backups

#### Overview

Like other modern mobile operating systems Android offers auto-backup features. The backups usually include copies of the data and settings of all apps installed on the device. An obvious concern is whether sensitive user data stored by the app might unintentionally leak to those data backups.

Given its diverse ecosystem, Android has a lot of backup options to account for.

- Stock Android has built-in USB backup facilities. A full data backup, or a backup of a particular app's data directory, can be obtained using the <code>adb backup</code> command when USB debugging is enabled.

- Google also provides a "Back Up My Data" feature that backs up all app data to Google's servers.

- Two Backup APIs are available to app developers:

  - [Key/Value Backup](https://developer.android.com/guide/topics/data/keyvaluebackup.html "Key/Value Backup") (Backup API or Android Backup Service) uploads selected data to the Android Backup Service.

  - [Auto Backup for Apps](https://developer.android.com/guide/topics/data/autobackup.html "Auto Backup for Apps"): With Android 6.0 (>= API level 23), Google added the "Auto Backup for Apps feature". This feature automatically syncs up to 25MB of app data to the user's Google Drive account.

- OEMs may add additional options. For example, HTC devices have a "HTC Backup" option that, when activated, performs daily backups to the cloud.

#### Static Analysis

##### Local

In order to backup all your application data Android provides an attribute called [`allowBackup`](https://developer.android.com/guide/topics/manifest/application-element.html#allowbackup "allowBackup attribute"). This attribute is set within the `AndroidManifest.xml` file. If the value of this attribute is set to **true**, then the device allows users to backup the application using Android Debug Bridge (ADB) via the command `$ adb backup`.

> Note: If the device was encrypted, then the backup files will be encrypted as well.

Check the `AndroidManifest.xml` file for the following flag:

```xml
android:allowBackup="true"
```

If the value is set to **true**, investigate whether the app saves any kind of sensitive data, check the test case "Testing for Sensitive Data in Local Storage".

##### Cloud
Regardless of using either key/value or auto backup, it needs to be identified:
- what files are sent to the cloud (e.g. SharedPreferences),
- if the files contain sensitive information and
- if sensitive information is protected through encryption before sending it to the cloud.

- **Auto Backup**: Auto Backup is configured through the boolean attribute `android:allowBackup` within the application's manifest file. If not explicitly set, applications targeting Android 6.0 (API Level 23) or higher enable [Auto Backup](https://developer.android.com/guide/topics/data/autobackup.html#EnablingAutoBackup "Enabling AutoBackup") by default. The attribute `android:fullBackupOnly` can also be used to activate auto backup when implementing a backup agent, but this is only available from Android 6.0 onwards. Other Android versions will be using key/value backup instead.

```xml
android:fullBackupOnly
```

Auto backup includes almost all of the app files and stores them in the Google Drive account of the user, limited to 25MB per app. Only the most recent backup is stored, the previous backup is deleted.

- **Key/Value Backup**: To enable key/value backup the backup agent needs to be defined in the manifest file. Look in `AndroidManifest.xml` for the following attribute:

```xml
android:backupAgent
```

To implement the key/ value backup, either one of the following classes needs to be extended:
- [BackupAgent](https://developer.android.com/reference/android/app/backup/BackupAgent.html "BackupAgent")
-  [BackupAgentHelper](https://developer.android.com/reference/android/app/backup/BackupAgentHelper.html "BackupAgentHelper")

Look for these classes within the source code to check for implementations of key/value backup.


#### Dynamic Analysis

After executing all available functions when using the app, attempt to make a backup using `adb`. If successful, inspect the backup archive for sensitive data. Open a terminal and run the following command:

```bash
$ adb backup -apk -nosystem <package-name>
```

Approve the backup from your device by selecting the _Back up my data_ option. After the backup process is finished, you will have a _.ab_ file in your current working directory.
Run the following command to convert the .ab file into a .tar file.

```bash
$ dd if=mybackup.ab bs=24 skip=1|openssl zlib -d > mybackup.tar
```

Alternatively, use the [_Android Backup Extractor_](https://github.com/nelenkov/android-backup-extractor "Android Backup Extractor") for this task. For the tool to work, you also have to download the Oracle JCE Unlimited Strength Jurisdiction Policy Files for [JRE7](http://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html "Oracle JCE Unlimited Strength Jurisdiction Policy Files JRE7") or [JRE8](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html "Oracle JCE Unlimited Strength Jurisdiction Policy Files JRE8"), and place them in the JRE lib/security folder. Run the following command to convert the tar file:

```bash
java -jar android-backup-extractor-20160710-bin/abe.jar unpack backup.ab
```

Extract the tar file into your current working directory to perform your analysis for sensitive data.

```bash
$ tar xvf mybackup.tar
```

#### Remediation

To prevent backing up the app data, set the `android:allowBackup` attribute to **false** in `AndroidManifest.xml`. If this attribute is not available the allowBackup setting is enabled by default. Therefore it needs to be explicitly set in order to deactivate it.

Sensitive information should not be sent in clear text to the cloud. Either,
- avoid storing the information in the first place, or
- encrypt the information at rest, before sending it to the cloud.

Files can also be excluded from [Auto Backup](https://developer.android.com/guide/topics/data/autobackup.html#IncludingFiles "Exclude files from Auto Backup"), in case they should not be shared with the Google Cloud.


#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage
- M2 - Insecure Data Storage

##### OWASP MASVS
- V2.8: "No sensitive data is included in backups generated by the mobile operating system."

##### CWE
- CWE-530 - Exposure of Backup File to an Unauthorized Control Sphere

##### Tools
- Android Backup Extractor - https://github.com/nelenkov/android-backup-extractor



### Testing for Sensitive Information in Auto-Generated Screenshots

#### Overview

Manufacturers want to provide device users an aesthetically pleasing effect when an application is entered or exited, hence they introduced the concept of saving a screenshot when the application goes into the background. This feature could potentially pose a security risk for an application. Sensitive data could be exposed if a user deliberately takes a screenshot of the application while sensitive data is displayed, or in the case of a malicious application running on the device, that is able to continuously capture the screen. This information is written to local storage, from which it may be recovered either by a rogue application on a rooted device, or by someone who steals the device.

For example, capturing a screenshot of a banking application running on the device may reveal information about the user account, his credit, transactions and so on.

#### Static Analysis

In Android, when the app goes into background a screenshot of the current activity is taken and is used to give a pleasing effect when the app is entered again. However, this would leak sensitive information that is present within the app.

To verify if the application may expose sensitive information via app switcher, detect if the [`FLAG_SECURE`](https://developer.android.com/reference/android/view/Display.html#FLAG_SECURE "FLAG_SECURE Option") option is set. You should be able to find something similar to the following code snippet.

```Java
LayoutParams.FLAG_SECURE
```

If not, the application is vulnerable to screen capturing.

#### Dynamic Analysis

During black-box testing, open any screen within the app that contains sensitive information and click on the home button so that the app goes into background. Now press the app-switcher button, to see the snapshot. As shown below, if `FLAG_SECURE` is set (image on the right), the snapshot is empty, while if the `FLAG_SECURE` is not set (image on the left), information within the activity is shown:

| `FLAG_SECURE` not set  | `FLAG_SECURE` set  |
|---|---|
| ![OMTG_DATAST_010_1_FLAG_SECURE](Images/Chapters/0x05d/1.png)   |  ![OMTG_DATAST_010_2_FLAG_SECURE](Images/Chapters/0x05d/2.png) |


#### Remediation

To prevent users or malicious applications from accessing information from backgrounded applications use the `FLAG_SECURE` as shown below:

```Java
getWindow().setFlags(WindowManager.LayoutParams.FLAG_SECURE,
                WindowManager.LayoutParams.FLAG_SECURE);

setContentView(R.layout.activity_main);
```

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage
- M2 - Insecure Data Storage

##### OWASP MASVS
- V2.9: "The app removes sensitive data from views when backgrounded."

##### CWE
- CWE-200 - Information Exposure


### Testing for Sensitive Data in Memory

#### Overview

Analyzing the memory can help to identify the root cause of different problems, like for example why an application is crashing, but can also be used to identify sensitive data. This section describes how to check for sensitive data and disclosure of data in general within the process memory.

To be able to investigate the memory of an application a memory dump needs to be created first or the memory needs to be viewed with real-time updates. This is also already the problem, as the application only stores certain information in memory if certain functions are triggered within the application. Memory investigation can of course be executed randomly in every stage of the application, but it is much more beneficial to understand first what the mobile app is doing and what kind of functionalities it offers and also make a deep dive into the (decompiled) source code before making any memory analysis.
Once sensitive functions are identified, like decryption of data, the investigation of a memory dump might be beneficial in order to identify sensitive data like a key or the decrypted information itself.

#### Static Analysis

First, you need to identify which sensitive information is stored in memory. Then there are a few checks that must be executed:

- Verify that no sensitive information is stored in an immutable structure. Immutable structures are not really overwritten in the heap, even after nullification or changing them. Instead, by changing the immutable structure, a copy is created on the heap. `BigInteger` and `String` are two of the most used examples when storing secrets in memory.
- Verify that, when mutable structures are used, such as `byte[]` and `char[]` that all copies of the structure are cleared.

-*NOTICE**: Destroying a key (e.g. `SecretKey secretKey = new SecretKeySpec("key".getBytes(), "AES"); secret.destroy();`) does *not* work, nor nullifying the backing byte-array from `secretKey.getEncoded()` as the SecretKeySpec based key returns a copy of the backing byte-array.
Therefore the developer should, in case of not using the `AndroidKeyStore` make sure that the key is wrapped and properly protected (see the remediation section for more details).
Understand that an RSA key pair is based on `BigInteger` as well and therefore reside in memory after first use outside of the `AndroidKeyStore`.
Lastly, some ciphers do not properly clean up their byte-arrays. For instance, the AES `Cipher` in `BouncyCastle` does not always clean up its latest working key.

#### Dynamic Analysis

For rudimentary analysis Android Studio built-in tools can be used. Android Studio includes tools in the _Android Monitor_ tab to investigate the memory. Select the device and app you want to analyze in the _Android Monitor_ tab and click on _Dump Java Heap_ and a _.hprof_ file will be created.

![Create Heap Dump](Images/Chapters/0x05d/Dump_Java_Heap.png)

In the new tab that shows the _.hprof_ file, the "Package Tree View" should be selected. Afterwards the package name of the app can be used to navigate to the instances of classes that were saved in the memory dump.

![Create Heap Dump](Images/Chapters/0x05d/Package_Tree_View.png)

The _.hprof_ file will be stored in the directory "captures", relative to the project path open within Android Studio. For deeper analysis of the memory dump the tool Eclipse Memory Analyzer (MAT) should be used.

Before the _.hprof_ file can be opened in MAT it needs to be converted. The tool _hprof-conf_ can be found in the Android SDK in the directory platform-tools.

```bash
./hprof-conv file.hprof file-converted.hprof
```

By using MAT, more functions are available, like usage of the Object Query Language (OQL). OQL is an SQL-like language that can be used to make queries in the memory dump. Analysis should be done on the dominator tree as only this contains the variables/memory of static classes.

To quickly discover potential sensitive data in the _.hprof_ file, it is also useful to run the `string` command against it. When doing a memory analysis, check for sensitive information like:
- Password and/or usernames
- Decrypted information
- User or session related information
- Interaction with OS, e.g. reading file content

#### Remediation

In Java, no immutable structures should be used to carry secrets (e.g. `String`, `BigInteger`). Nullifying them will not be effective: the garbage collector might collect them, but they might remain in the JVM's heap for a longer period.
Rather use byte-arrays (`byte[]`) or char-arrays (`char[]`) which are cleaned after the operations are done:


```java

byte[] secret = null;
try{
	//get or generate the secret, do work with it, make sure you make no local copies
} finally {
	if (null != secret && secret.length > 0) {
		for (int i = 0; i < secret; i++) {
			array[i] = (byte) 0;
		}
	}
}
```

Also look into the best practices for [securely storing sensitive data in RAM](https://www.nowsecure.com/resources/secure-mobile-development/coding-practices/securely-store-sensitive-data-in-ram/ "Securely store sensitive data in RAM")

Keys should be handled by the `AndroidKeyStore` or the `SecretKey` class needs to be adjusted. For a better implementation of the `SecretKey` one can use the `ErasableSecretKey` class below. This class consists of two parts:
- A wrapper class called `ErasableSecretKey` which takes care of building up the internal key, adding a clean method and a static convenience method. You can call the `getKey()` on a `ErasableSecretKey` to get the actual key.
- An internal `InternalKey` class which implements `javax.crypto.SecretKey, Destroyable`, so you can actually destroy it and it will behave as a SecretKey from JCE. The destroyable implementation first sets null bytes to the internal key and then it will put null as a reference to the byte[] representing the actual key. As you can see the `InternalKey` does not provide a copy of its internal byte[] representation, instead it gives the actual version. This will make sure that you will no longer have copies of the key in many parts of your application memory.


```java
public class ErasableSecretKey implements Serializable {

    public static final int KEY_LENGTH = 256;

    private java.security.Key secKey;

	// Do not try to instantiate it: use the static methods.
	// The static construction methods only use mutable structures or create a new key directly.
    protected ErasableSecretKey(final java.security.Key key) {
        this.secKey = key;
    }

	//Create a new `ErasableSecretKey` from a byte-array.
	//Don't forget to clean the byte-array when you are done with the key.
    public static ErasableSecretKey fromByte(byte[] key) {
        return new ErasableSecretKey(new SecretKey.InternalKey(key, "AES"));
    }
	//Create a new key. Do not forget to implement your own 'Helper.getRandomKeyBytes()'.
    public static ErasableSecretKey newKey() {
        return fromByte(Helper.getRandomKeyBytes());
    }

	//clean the internal key, but only do so if it is not destroyed yet.
    public void clean() {
        try {
            if (this.getKey() instanceof Destroyable) {
                ((Destroyable) this.getKey()).destroy();
            }

        } catch (DestroyFailedException e) {
            //choose what you want to do now: so you could not destroy it, would you run on? Or rather inform the caller of the clean method informing him of the failure?
        }
    }
	//convenience method that takes away the null-check so you can always just call ErasableSecretKey.clearKey(thekeytobecleared)
    public static void clearKey(ErasableSecretKey key) {
        if (key != null) {
            key.clean();
        }
    }

	//internal key class which represents the actual key.
    private static class InternalKey implements javax.crypto.SecretKey, Destroyable {
        private byte[] key;
        private final String algorithm;

        public InternalKey(final byte[] key, final String algorithm) {
            this.key = key;
            this.algorithm = algorithm;
        }

        public String getAlgorithm() {
            return this.algorithm;
        }

        public String getFormat() {
            return "RAW";
        }

		//Do not return a copy of the byte-array but the byte-array itself. Be careful: clearing this byte-array, will clear the key.
        public byte[] getEncoded() {
            if(null == this.key){
               throw new NullPointerException();
            }
            return this.key;
        }

		//destroy the key.
        public void destroy() throws DestroyFailedException {
            if (this.key != null) {
                Arrays.fill(this.key, (byte) 0);
            }

            this.key = null;
        }

        public boolean isDestroyed() {
            return this.key == null;
        }
    }


    public final java.security.Key getKey() {
        return this.secKey;
    }

}

```

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage
- M2 - Insecure Data Storage

##### OWASP MASVS
- V2.10: "The app does not hold sensitive data in memory longer than necessary, and memory is cleared explicitly after use."

##### CWE
- CWE-316 - Cleartext Storage of Sensitive Information in Memory

##### Tools
- Memory Monitor - http://developer.android.com/tools/debugging/debugging-memory.html#ViewHeap
- Eclipse’s MAT (Memory Analyzer Tool) standalone - https://eclipse.org/mat/downloads.php
- Memory Analyzer which is part of Eclipse - https://www.eclipse.org/downloads/
- Fridump - https://github.com/Nightbringer21/fridump
- LiME - https://github.com/504ensicsLabs/LiME


### Testing the Device-Access-Security Policy

#### Overview

Apps that are processing or querying sensitive information should ensure that they are running in a trusted and secured environment. In order to be able to achieve this, the app can enforce the following local checks on the device:

- PIN or password set to unlock the device
- Usage of a minimum Android OS version
- Detection of activated USB Debugging
- Detection of encrypted device
- Detection of rooted device (see also "Testing Root Detection")

#### Static Analysis

In order to be able to test the device-access-security policy that is enforced by the app, a written copy of the policy needs to be provided. The policy should define what checks are available and how they are enforced. For example one check could require that the app only runs on Android Marshmallow (Android 6.0) or higher and the app is closing itself or showing a warning if the app is running on an Android version < 6.0.

The functions within the code that implement the policy need to be identified and checked if they can be bypassed.

#### Dynamic Analysis

The dynamic analysis depends on the checks that are enforced by app and their expected behavior and need to be validated if they can be bypassed.

#### Remediation

Different checks on the Android device can be implemented by querying different system preferences from [_Settings.Secure_](https://developer.android.com/reference/android/provider/Settings.Secure.html "Settings.Secure"). The [_Device Administration API_](https://developer.android.com/guide/topics/admin/device-admin.html "Device Administration API") offers different mechanisms to create security aware applications, that are able to enforce password policies or encryption of the device.

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage

##### OWASP MASVS
- V2.11: "The app enforces a minimum device-access-security policy, such as requiring the user to set a device passcode."



### Verifying User Education Controls

#### Overview

Educating users is a crucial part in the usage of mobile apps. Even though many security controls are already in place, they might be circumvented or misused through the user.

The following list shows potential warnings or advises for a user when opening the app the first time and using it:
- Showing a list of what kind of data is stored locally and remotely. This can also be a link to an external resource as the information might be quite extensive.
- If a new user account is created within the app it should show the user if the password provided is considered secure and applies to the password policy.
- If the user is installing the app on a rooted device a warning should be shown that this is dangerous and deactivates security controls at OS level and is more likely to be prone to malware. See also "Testing Root Detection" for more details.
- If a user installed the app on an outdated Android version a warning should be shown. See also "Testing the Device-Access-Security Policy" for more details.

#### Static Analysis

A list of implemented education controls should be provided. The controls should be verified in the code if they are implemented properly and according to best practices.

#### Dynamic Analysis

After installing the app and also while using it, it should be checked if any warnings are shown to the user, that have an educational purpose and are aligned with the defined education controls.

#### Remediation

Warnings should be implemented that address the key points listed in the overview section.

#### References

##### OWASP Mobile Top 10 2016
- M1 - Improper Platform Usage

##### OWASP MASVS
- V2.12: "The app educates the user about the types of personally identifiable information processed, as well as security best practices the user should follow in using the app."
