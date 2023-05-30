# Workflow guidelines

* When contributing a PullRequest, separate your commits in units of work (don't mix changes that have different concerns in the same commit). Don't forget to include all explanations and reasonings in the commit messages, instead of just leaving them as part of the PullRequest description.

* The most common mistake when mixing multiple concerns in the same commit is to join both refactorings and change of behaviour in the same change. As a convention, we will consider the word "refactoring" to refer to changes in the code that should not have any impact on the general behaviour of the program (especially talking about behaviour noticed by the end-user).

* Push each commit separately (instead of sending more than 1 commit in a single push), so that we can have a CI status for each commit in the PullRequest. This is a best practice because it will make sure that the build is not broken in between commits (otherwise, future developers may have a hard time when trying to bisect bugs). If you have already pushed your commits to the remote in one push, this can be re-done by using our [gitPush1by1.fsx](https://github.com/nblockchain/conventions/blob/master/scripts/gitPush1by1.fsx) script, or this technique manually: https://stackoverflow.com/a/3230241/544947

* Group import declarations (e.g. `open` in F# and `using` in C#) in three buckets:
    * The first group for the namespaces that come from the base class libraries.
    * Second group for external libraries (e.g. nuget packages).
    * Last for the namespaces that belong to the source code that lives in same repository.

* Avoid typical bad practices like:

    * Magic numbers:

      Avoid using unnamed numerical constants in software code, this practice makes code hard to understand and maintain.

      Example (with bad practice):
      ```csharp
      var distance = GpsUtil.GetDistance();
      if (distance < 100)
         throw new NotImplementedException();
      ```

      Improved code:
      ```csharp
      private const int MinimumSupportedDistanceToNotifyKillerDrones = 100;

      ...

      var distance = GpsUtil.GetDistance()
      if (distance < MinimumSupportedDistanceToNotifyKillerDrones)
         throw new NotImplementedException();
      ```

    * DRY (Don't Repeat Yourself):

      The DRY principle suggests that a piece of information should only be stored once in a project and should be referenced as needed, rather than being copied and pasted multiple times throughout the codebase.

      It has several benefits, including reducing the amount of code that needs to be written and maintained, improving the consistency and quality of the code, and reducing the risk of introducing errors and bugs when the information changes.

      Example (with bad practice):
      ```fsharp
      let preAuthInputMac =
         CalculateMacWithSHA3256
             preAuthInput
             ":hs_mac"

      ...

      let authInputMac =
         CalculateMacWithSHA3256
             authInput
             ":hs_mac"
      ```

      Improved code:
      ```fsharp
      let AuthenticationDigestCalculationKey = ":hs_mac"

      ...

      let preAuthInputMac =
         CalculateMacWithSHA3256
             preAuthInput
             AuthenticationDigestCalculationKey

      ...

      let authInputMac =
         CalculateMacWithSHA3256
             authInput
             AuthenticationDigestCalculationKey
      ```

    * Primitive Obsession:

      Primitive Obsession is a situation where simple data types such as strings, integers, or arrays are overused in place of more appropriate objects.
      
      Example (with bad practice):
      ```
      let saveFilePath = System.Console.ReadLine()
      
      let savedData = System.IO.File.ReadAllText saveFilePath
      ```

      Improved code:
      ```fsharp
      let saveFilePath =
         let saveFilePathInString =
             System.Console.ReadLine()
         System.IO.FileInfo saveFilePathInString

      let savedData = System.IO.File.ReadAllText saveFilePath.FullName
      ```

    * Discarding generic exceptions:

        Discarding generic exceptions is a bad practice because it can lead to unexpected behavior and bugs. It's better to handle the exception using a non-generic catch, or at least log them to help with debugging if something unexpected happens.
    
        Example (with bad practice):
        ```fsharp
        let Func (zipFile: FileInfo) =
            try
                ZipFile.Open(zipFile.FullName, ZipArchiveMode.Read) |> ignore<ZipArchive>
            with
            | _ -> 
                ()
        ```

        Improved code:
        ```fsharp
        let Func (zipFile: FileInfo) =
            try
                ZipFile.Open(zipFile.FullName, ZipArchiveMode.Read) |> ignore<ZipArchive>
            with
            | :? InvalidDataException -> 
                Console.Error.WriteLine $"Not a zip file %s{zipFile.FullName}"
            | :? FileNotFoundException -> 
                Console.Error.WriteLine $"File not found %s{zipFile.FullName}"
            | _ -> 
                reraise()
        ```

    * Catching preventable exceptions:

        There are exceptions that cannot really be predicted from the caller of the function and to handle them there is no other way than using `catch` (C#) or `with` (F#) blocks. However, there are exceptions that are completely preventable and should never be caught:

        Example (with bad practice):
        ```fsharp
            try
                name.ComposeFullName(separator)
            with
            | :? ArgumentNullException ->
                Console.Error.WriteLine "There was a problem retreiving the name"
            | :? NullReferenceException ->
                Console.Error.WriteLine "There was a problem in the naming system, please report this bug"
        ```

        Improved code:
        ```fsharp
            if isNull name then
                Console.Error.WriteLine "There was a problem in the naming system, please report this bug"
            elif isNull separator then
                Console.Error.WriteLine "There was a problem retreiving the name"
            else
                name.ComposeFullName separator
        ```

        This way, the code is more performant (even though this is the least important aspect), readable (as we can immediately see what is the potentially problematic variable) and maintainable (as new unexpected but unrelated exceptions can be generated by future versions of the library that contains the function called).

* If you want to contribute a script, do not use PowerShell or Bash, but
an F# script. The reason to not use PowerShell is a personal preference
from the maintainer of this project (and his suspicion that it might not
be 100% guaranteed to be crossplatform); and the reason not to use Bash
is because it's only Unix compatible (we cannot assume WSL is installed
in Windows), and in general because it's too undeterministic and old, more
info here: https://news.ycombinator.com/item?id=33116310
* Don't add obvious comments that can be inferred from just reading the code;
instead, use comments to explain why you're doing something, not what the
code is doing. Sometimes you can even extract some piece of code as a separate
function and name the function in such a way that it explains what the code
is doing and therefore there's no need to add a comment anymore.
* Add comments on top of code (on a line above it), not next to it, to avoid
horizontal scrolling.
* Do not commit commented code; if the code is commented it's better that it
gets removed. If there's a reason for why the commented piece of code should
not be removed, then write the reason why, in a comment on top of it. Otherwise
it's extremely confusing for the next developer (which could be your future
you) to find code that is commented/disabled.
* Our naming conventions are as follows:
    * Script names (e.g. files with `.sh`, `.bat` or `.fsx` extensions): snake_case.
    * CI job names: kebab-case.
    * Important public constants: TRAIN_CASE.
    * .NET (F# and C#) source files and projects (project names and project file names): PascalCase. An exception to this rule is project names and project file names that correspond to a console project which 
ends up being compiled and packaged in NuGet as a dotnet tool, for example: https://github.com/nblockchain/fsx/tree/master/fsxc
    * .NET APIs: PascalCase (see our [F# Style Guide](FSharpStyleGuide.md) for more info).
    * .NET parameters, local variables & nested functions: camelCase (again, see our [F# Style Guide](FSharpStyleGuide.md) for more info).
* Git commit messages should follow this style:

```
Scope/sub-scope: short title of what is changed (50 chars max)

Explanation of **why** (and maybe **how** as well, in case there's a part of
the change that is not self-explanatory). Don't hesitate to be very verbose
here, adding any references you may need, in this way[1], or even @nicknames of
people that helped. Manually crop your lines to not be longer than 64 chars.

Fixes https://github.com/nblockchain/geewallet/issues/45

[1] http://foo.bar/baz
```

As you can see, writing a commit message is generally like writing an e-mail: it
has a title at the top which is normally a short sentence (but not ended with a
dot, like most titles), and a body that starts in the 3rd line and which contains
one or many paragraphs (each ending with a dot, as it's text in prose). In
particular, the example above would be for a commit message that fixes the
issue #45. **Scope** usually refers to the project name, but without the need
to include the name of the project (e.g. in geewallet, all project names start
with the `GWallet.` prefix, then there's no need to specify it; so use `Backend`
as scope instead of `GWallet.Backend`). The **Sub-scope** may refer to a folder
or module inside what's represented as the scope, but it's not a strict mapping.

When referencing a bug/issue, as you can see above you can add a sentence at the
end of the commit message which starts with `Fixes `, followed by the **full URL**
of the bug/issue; this way, the ticket will be closed when the commit lands in
the main branch (which in some repos will be called `master`, or `main`). If the
commit is not really a fix for the issue, but you still want the ticket to be
closed after the commit lands, then you would use the word `Closes` instead of
`Fixes`.

Do not use long lines (manually crop them with EOLs because git doesn't do this
automatically).

