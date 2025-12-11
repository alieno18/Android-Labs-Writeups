We run the app and see that the user is prompted to allow the access to manage all files, we allow the access, mimicking what an user would do with their antivirus software:
![](./assets/Cyclic%20Scanner/20251122160456805-Cyclic%20Scanner.png)

Then we see a button which seem to enable the scanner:

![](./assets/Cyclic%20Scanner/20251122160644942-Cyclic%20Scanner.png)

We decompile the apk with jadx and see that in the MainActivity class there is the code to manage behaviour when a button is changed. In particular, when the button is checked a service starts whose class is `ScanService`:

![](./assets/Cyclic%20Scanner/20251121193341759-Cyclic%20Scanner.png)

Therefore, we check the `ScanService` file, where are define the `onCreate` and `onStartCommand`, which are default in android services. In particular, in `onStartCommand` an istance of `serviceHandler` is defined and the method `sendMessage` is called.

![](./assets/Cyclic%20Scanner/20251122162646214-Cyclic%20Scanner.png)

The `serviceHandler` extends `Handler`, and after the `sendMessage` method is called the `handleMessage` is called with the same message:

![](./assets/Cyclic%20Scanner/20251122162459567-Cyclic%20Scanner.png)

In the method `handleMessage`, a `scanEngine` instance is defined and the method `scanFile` is called. Based on a boolean result it is print SAFE or INFECTED:

![](./assets/Cyclic%20Scanner/20251122162855424-Cyclic%20Scanner.png)

Therefore, we check the method `scanFile` defined in the class `ScanEngine`:

![](./assets/Cyclic%20Scanner/20251122163432445-Cyclic%20Scanner.png)

This method as a serious vulnerability, indeed:
- `command` → a string defined in the first row
- `sh` → runs a shell
- `-c` → tells the shell to execute the string `command`

Hence, it executes an apparently safe string as command by calling the shell, the string is:

```
toybox sha1sum <absolute_path_of_scanned_file>
```

The file are taken iteratively with these lines in `handleMessage` and then passed to the `scanFile` method:

![](./assets/Cyclic%20Scanner/20251122164041724-Cyclic%20Scanner.png)

Basically, the whole `/sdcard` (or if you want `/storage/emulate/0`) is scanned recursively.

The vulnerability here is that if a file has maliciously crafted name and put in that directory, it is possible to execute arbitrary command (restricted by app permissions) and achieve RCE.

To show this we create a file using the symbol ";" which in bash allows to concatenates two commands. We craft this file with this command:

```
echo "not_important" > "placeholder;log IF_YOU_SEE_THIS_RCE_IS_REAL \$(id)"
```

This should create a file with filename:

```
placeholder;log IF_YOU_SEE_THIS_RCE_IS_REAL \$(id)
```

and when the scan get this filename it will execute two commands:

```
toybox sha1sum placeholder
log IF_YOU_SEE_THIS_RCE_IS_REAL \$(id)
```

Let us attempt the exploit, we activate the scanner:

![](./assets/Cyclic%20Scanner/20251122165931935-Cyclic%20Scanner.png)


![](./assets/Cyclic%20Scanner/20251122165817968-Cyclic%20Scanner.png)

The command is injected correctly and thus we achieved RCE.