# Unpacking OriginLogger Builder

OriginLogger is a keylogger that shares a lot of similarities with the well known Agent Tesla malware. Today I will take a look at their builder and unpack it. A little spoiler the sample used in this post was protected with a trial version of the commercial obfuscator Eazfuscator.NET which stops the binary from running due to the expired trial version...

You can find the sample on [malshare](https://malshare.com/sample.php?action=detail&hash=595a7ea981a3948c4f387a5a6af54a70a41dd604685c72cbd2a55880c2b702ed)


# Initial Analysis

Opening the sample in dnSpy will show a lot of errors in the decompilation. Looking at the IL will reveal some invalid code. This is an indicator for method body encryption. Looking at the module constructor verifies that assumption. The first method called by the constructor at token `0x06000006` is responsible for decrypting the method bodies.

![constructor](/images/origin_decrypt.png)

Looking at the method we can quickly deduce what is approximately doing. It starts with obtaining the `HINSTANCE` of the current module. Which is a pointer to the base of the currently loaded module aka the PE header base. To that pointer it adds `0x3C` which points to `e_lfanew` a field in the PE header which holds the file address of the new executable header. The value of `e_lfanew` is added to the base pointer and then used to read another value by adding offset `0x06` which points to `NumberOfSections`. The number of sections is stored in a local for later use. Next it reads from offset `0x14` which points to `SizeOfOptionalHeader` this is important since the size of this part of the PE header varies depending on bitness. Using the size and offset `0x18` which points to the beginning of the section table. When it finds In the following the code loops trough the sections to find its abritary section in which the encrypted method bodies are stored. It will then decrypt the method bodies. (I shortened this explanation due to time constraints)

## Dumping

In order to get the unecrypted method bodies I will debug and dump the file. We simply let the decryption method run and place a breakpoint on the second call in the module constructor.
We can safely ignore the anti debug code in the method decryption routine since dnSpy by default hooks `CheckRemoteDebuggerPresent` and `IsDebuggerPresent` to avoid detection.

![breakpoint](/images/origin_constructor.png)

Once the breakpoint hits open the Modules tab and right click the main module. Save it to disk.

![dumping](/images/origin_dump.png)

Now open the saved module and patch both calls in the module constructor with a nop instruction, since we dont need to decrypt methods anymore and the second call is runtime anti dump which we also dont need. Save the changes.

## Deobfuscating

Next we will use [de4dot](https://github.com/de4dot/de4dot) to get rid of the unicode names. Simply drag & drop the dumped and patched binary into de4dot and let it do its work. After de4dot has finnished we are left with the string encryption for that we will use a tool called [eazfix](https://github.com/HoLLy-HaCKeR/EazFixer) make sure to use the `--keep-types` argument. Eazfix will decrypt the strings for us, if youre curious how it works it uses Harmony to patch the stackframe calls used by Eazfuscators string encryption. After patching the stackframe method to always return the string decryption method it can simply invoke the string decryption routine for each string and patch the call with the resulting string.

After eazfixer is done you have a fully unpacked Origin Loader sample ;)

