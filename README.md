#  Solidity
v0.5.7
Search docs
Introduction to Smart Contracts
Installing the Solidity Compiler
Solidity by Example
Solidity in Depth
Security Considerations
Resources
Using the compiler
Using the Commandline Compiler
Setting the EVM version to target
Compiler Input and Output JSON Description
Input Description
Output Description
Contract Metadata
Contract ABI Specification
Yul
Style Guide
Common Patterns
List of Known Bugs
Contributing
LLL
Keyword Index
Solidity
Docs » Using the compiler
Using the compiler
Using the Commandline Compiler
Note

This section does not apply to solcjs, not even if it is used in commandline mode.

One of the build targets of the Solidity repository is solc, the solidity commandline compiler. Using solc --help provides you with an explanation of all options. The compiler can produce various outputs, ranging from simple binaries and assembly over an abstract syntax tree (parse tree) to estimations of gas usage. If you only want to compile a single file, you run it as solc --bin sourceFile.sol and it will print the binary. If you want to get some of the more advanced output variants of solc, it is probably better to tell it to output everything to separate files using solc -o outputDirectory --bin --ast --asm sourceFile.sol.

Before you deploy your contract, activate the optimizer when compiling using solc --optimize --bin sourceFile.sol. By default, the optimizer will optimize the contract assuming it is called 200 times across its lifetime (more specifically, it assumes each opcode is executed around 200 times). If you want the initial contract deployment to be cheaper and the later function executions to be more expensive, set it to --optimize-runs=1. If you expect many transactions and do not care for higher deployment cost and output size, set --optimize-runs to a high number. This parameter has effects on the following (this might change in the future):

the size of the binary search in the function dispatch routine
the way constants like large numbers or strings are stored
The commandline compiler will automatically read imported files from the filesystem, but it is also possible to provide path redirects using prefix=path in the following way:

solc github.com/ethereum/dapp-bin/=/usr/local/lib/dapp-bin/ file.sol
This essentially instructs the compiler to search for anything starting with github.com/ethereum/dapp-bin/ under /usr/local/lib/dapp-bin. solc will not read files from the filesystem that lie outside of the remapping targets and outside of the directories where explicitly specified source files reside, so things like import "/etc/passwd"; only work if you add /=/ as a remapping.

An empty remapping prefix is not allowed.

If there are multiple matches due to remappings, the one with the longest common prefix is selected.

For security reasons the compiler has restrictions what directories it can access. Paths (and their subdirectories) of source files specified on the commandline and paths defined by remappings are allowed for import statements, but everything else is rejected. Additional paths (and their subdirectories) can be allowed via the --allow-paths /sample/path,/another/sample/path switch.

If your contracts use libraries, you will notice that the bytecode contains substrings of the form __$53aea86b7d70b31448b230b20ae141a537$__. These are placeholders for the actual library addresses. The placeholder is a 34 character prefix of the hex encoding of the keccak256 hash of the fully qualified library name. The bytecode file will also contain lines of the form // <placeholder> -> <fq library name> at the end to help identify which libraries the placeholders represent. Note that the fully qualified library name is the path of its source file and the library name separated by :. You can use solc as a linker meaning that it will insert the library addresses for you at those points:

Either add --libraries "file.sol:Math:0x1234567890123456789012345678901234567890 file.sol:Heap:0xabCD567890123456789012345678901234567890" to your command to provide an address for each library or store the string in a file (one library per line) and run solc using --libraries fileName.

If solc is called with the option --link, all input files are interpreted to be unlinked binaries (hex-encoded) in the __$53aea86b7d70b31448b230b20ae141a537$__-format given above and are linked in-place (if the input is read from stdin, it is written to stdout). All options except --libraries are ignored (including -o) in this case.

If solc is called with the option --standard-json, it will expect a JSON input (as explained below) on the standard input, and return a JSON output on the standard output. This is the recommended interface for more complex and especially automated uses.

Note

The library placeholder used to be the fully qualified name of the library itself instead of the hash of it. This format is still supported by solc --link but the compiler will no longer output it. This change was made to reduce the likelihood of a collision between libraries, since only the first 36 characters of the fully qualified library name could be used.

Setting the EVM version to target
When you compile your contract code you can specify the Ethereum virtual machine version to compile for to avoid particular features or behaviours.

Warning

Compiling for the wrong EVM version can result in wrong, strange and failing behaviour. Please ensure, especially if running a private chain, that you use matching EVM versions.

On the command line, you can select the EVM version as follows:

solc --evm-version <VERSION> contract.sol
In the standard JSON interface, use the "evmVersion" key in the "settings" field:

{
  "sources": { ... },
  "settings": {
    "optimizer": { ... },
    "evmVersion": "<VERSION>"
  }
}
Target options
Below is a list of target EVM versions and the compiler-relevant changes introduced at each version. Backward compatibility is not guaranteed between each version.

homestead (oldest version)
tangerineWhistle
Gas cost for access to other accounts increased, relevant for gas estimation and the optimizer.
All gas sent by default for external calls, previously a certain amount had to be retained.
spuriousDragon
Gas cost for the exp opcode increased, relevant for gas estimation and the optimizer.
byzantium
Opcodes returndatacopy, returndatasize and staticcall are available in assembly.
The staticcall opcode is used when calling non-library view or pure functions, which prevents the functions from modifying state at the EVM level, i.e., even applies when you use invalid type conversions.
It is possible to access dynamic data returned from function calls.
revert opcode introduced, which means that revert() will not waste gas.
constantinople
Opcodes create2`, ``extcodehash, shl, shr and sar are available in assembly.
Shifting operators use shifting opcodes and thus need less gas.
petersburg (default)
The compiler behaves the same way as with constantinople.
Compiler Input and Output JSON Description
The recommended way to interface with the Solidity compiler especially for more complex and automated setups is the so-called JSON-input-output interface. The same interface is provided by all distributions of the compiler.

The fields are generally subject to change, some are optional (as noted), but we try to only make backwards compatible changes.

The compiler API expects a JSON formatted input and outputs the compilation result in a JSON formatted output.

The following subsections describe the format through an example. Comments are of course not permitted and used here only for explanatory purposes.

Input Description
{
  // Required: Source code language. Currently supported are "Solidity" and "Yul".
  "language": "Solidity",
  // Required
  "sources":
  {
    // The keys here are the "global" names of the source files,
    // imports can use other files via remappings (see below).
    "myFile.sol":
    {
      // Optional: keccak256 hash of the source file
      // It is used to verify the retrieved content if imported via URLs.
      "keccak256": "0x123...",
      // Required (unless "content" is used, see below): URL(s) to the source file.
      // URL(s) should be imported in this order and the result checked against the
      // keccak256 hash (if available). If the hash doesn't match or none of the
      // URL(s) result in success, an error should be raised.
      // Using the commandline interface only filesystem paths are supported.
      // With the JavaScript interface the URL will be passed to the user-supplied
      // read callback, so any URL supported by the callback can be used.
      "urls":
      [
        "bzzr://56ab...",
        "ipfs://Qma...",
        "/tmp/path/to/file.sol"
        // If files are used, their directories should be added to the command line via
        // `--allow-paths <path>`.
      ]
    },
    "mortal":
    {
      // Optional: keccak256 hash of the source file
      "keccak256": "0x234...",
      // Required (unless "urls" is used): literal contents of the source file
      "content": "contract mortal is owned { function kill() { if (msg.sender == owner) selfdestruct(owner); } }"
    }
  },
  // Optional
  "settings":
  {
    // Optional: Sorted list of remappings
    "remappings": [ ":g/dir" ],
    // Optional: Optimizer settings
    "optimizer": {
      // disabled by default
      "enabled": true,
      // Optimize for how many times you intend to run the code.
      // Lower values will optimize more for initial deployment cost, higher values will optimize more for high-frequency usage.
      "runs": 200,
      // Switch optimizer components on or off in detail.
      // The "enabled" switch above provides two defaults which can be
      // tweaked here. If "details" is given, "enabled" can be omitted.
      "details": {
        // The peephole optimizer is always on if no details are given, use details to switch it off.
        "peephole": true,
        // The unused jumpdest remover is always on if no details are given, use details to switch it off.
        "jumpdestRemover": true,
        // Sometimes re-orders literals in commutative operations.
        "orderLiterals": false,
        // Removes duplicate code blocks
        "deduplicate": false,
        // Common subexpression elimination, this is the most complicated step but
        // can also provide the largest gain.
        "cse": false,
        // Optimize representation of literal numbers and strings in code.
        "constantOptimizer": false,
        // The new Yul optimizer. Mostly operates on the code of ABIEncoderV2.
        // It can only be activated through the details here.
        // This feature is still considered experimental.
        "yul": false,
        // Tuning options for the Yul optimizer.
        "yulDetails": {
          // Improve allocation of stack slots for variables, can free up stack slots early.
          // Activated by default if the Yul optimizer is activated.
          "stackAllocation": true
        }
      }
    },
    "evmVersion": "byzantium", // Version of the EVM to compile for. Affects type checking and code generation. Can be homestead, tangerineWhistle, spuriousDragon, byzantium, constantinople or petersburg
    // Metadata settings (optional)
    "metadata": {
      // Use only literal content and not URLs (false by default)
      "useLiteralContent": true
    },
    // Addresses of the libraries. If not all libraries are given here, it can result in unlinked objects whose output data is different.
    "libraries": {
      // The top level key is the the name of the source file where the library is used.
      // If remappings are used, this source file should match the global path after remappings were applied.
      // If this key is an empty string, that refers to a global level.
      "myFile.sol": {
        "MyLib": "0x123123..."
      }
    }
    // The following can be used to select desired outputs based
    // on file and contract names.
    // If this field is omitted, then the compiler loads and does type checking,
    // but will not generate any outputs apart from errors.
    // The first level key is the file name and the second level key is the contract name.
    // An empty contract name is used for outputs that are not tied to a contract
    // but to the whole source file like the AST.
    // A star as contract name refers to all contracts in the file.
    // Similarly, a star as a file name matches all files.
    // To select all outputs the compiler can possibly generate, use
    // "outputSelection: { "*": { "*": [ "*" ], "": [ "*" ] } }"
    // but note that this might slow down the compilation process needlessly.
    //
    // The available output types are as follows:
    //
    // File level (needs empty string as contract name):
    //   ast - AST of all source files
    //   legacyAST - legacy AST of all source files
    //
    // Contract level (needs the contract name or "*"):
    //   abi - ABI
    //   devdoc - Developer documentation (natspec)
    //   userdoc - User documentation (natspec)
    //   metadata - Metadata
    //   ir - Yul intermediate representation of the code before optimization
    //   irOptimized - Intermediate representation after optimization
    //   evm.assembly - New assembly format
    //   evm.legacyAssembly - Old-style assembly format in JSON
    //   evm.bytecode.object - Bytecode object
    //   evm.bytecode.opcodes - Opcodes list
    //   evm.bytecode.sourceMap - Source mapping (useful for debugging)
    //   evm.bytecode.linkReferences - Link references (if unlinked object)
    //   evm.deployedBytecode* - Deployed bytecode (has the same options as evm.bytecode)
    //   evm.methodIdentifiers - The list of function hashes
    //   evm.gasEstimates - Function gas estimates
    //   ewasm.wast - eWASM S-expressions format (not supported at the moment)
    //   ewasm.wasm - eWASM binary format (not supported at the moment)
    //
    // Note that using a using `evm`, `evm.bytecode`, `ewasm`, etc. will select every
    // target part of that output. Additionally, `*` can be used as a wildcard to request everything.
    //
    "outputSelection": {
      "*": {
        "*": [
          "metadata", "evm.bytecode" // Enable the metadata and bytecode outputs of every single contract.
          , "evm.bytecode.sourceMap" // Enable the source map output of every single contract.
        ],
        "": [
          "ast" // Enable the AST output of every single file.
        ]
      },
      // Enable the abi and opcodes output of MyContract defined in file def.
      "def": {
        "MyContract": [ "abi", "evm.bytecode.opcodes" ]
      }
    }
  }
}
Output Description
{
  // Optional: not present if no errors/warnings were encountered
  "errors": [
    {
      // Optional: Location within the source file.
      "sourceLocation": {
        "file": "sourceFile.sol",
        "start": 0,
        "end": 100
      ],
      // Mandatory: Error type, such as "TypeError", "InternalCompilerError", "Exception", etc.
      // See below for complete list of types.
      "type": "TypeError",
      // Mandatory: Component where the error originated, such as "general", "ewasm", etc.
      "component": "general",
      // Mandatory ("error" or "warning")
      "severity": "error",
      // Mandatory
      "message": "Invalid keyword"
      // Optional: the message formatted with source location
      "formattedMessage": "sourceFile.sol:100: Invalid keyword"
    }
  ],
  // This contains the file-level outputs. In can be limited/filtered by the outputSelection settings.
  "sources": {
    "sourceFile.sol": {
      // Identifier of the source (used in source maps)
      "id": 1,
      // The AST object
      "ast": {},
      // The legacy AST object
      "legacyAST": {}
    }
  },
  // This contains the contract-level outputs. It can be limited/filtered by the outputSelection settings.
  "contracts": {
    "sourceFile.sol": {
      // If the language used has no contract names, this field should equal to an empty string.
      "ContractName": {
        // The Ethereum Contract ABI. If empty, it is represented as an empty array.
        // See https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI
        "abi": [],
        // See the Metadata Output documentation (serialised JSON string)
        "metadata": "{...}",
        // User documentation (natspec)
        "userdoc": {},
        // Developer documentation (natspec)
        "devdoc": {},
        // Intermediate representation (string)
        "ir": "",
        // EVM-related outputs
        "evm": {
          // Assembly (string)
          "assembly": "",
          // Old-style assembly (object)
          "legacyAssembly": {},
          // Bytecode and related details.
          "bytecode": {
            // The bytecode as a hex string.
            "object": "00fe",
            // Opcodes list (string)
            "opcodes": "",
            // The source mapping as a string. See the source mapping definition.
            "sourceMap": "",
            // If given, this is an unlinked object.
            "linkReferences": {
              "libraryFile.sol": {
                // Byte offsets into the bytecode. Linking replaces the 20 bytes located there.
                "Library1": [
                  { "start": 0, "length": 20 },
                  { "start": 200, "length": 20 }
                ]
              }
            }
          },
          // The same layout as above.
          "deployedBytecode": { },
          // The list of function hashes
          "methodIdentifiers": {
            "delegate(address)": "5c19a95c"
          },
          // Function gas estimates
          "gasEstimates": {
            "creation": {
              "codeDepositCost": "420000",
              "executionCost": "infinite",
              "totalCost": "infinite"
            },
            "external": {
              "delegate(address)": "25000"
            },
            "internal": {
              "heavyLifting()": "infinite"
            }
          }
        },
        // eWASM related outputs
        "ewasm": {
          // S-expressions format
          "wast": "",
          // Binary format (hex string)
          "wasm": ""
        }
      }
    }
  }
}
Error types
JSONError: JSON input doesn’t conform to the required format, e.g. input is not a JSON object, the language is not supported, etc.
IOError: IO and import processing errors, such as unresolvable URL or hash mismatch in supplied sources.
ParserError: Source code doesn’t conform to the language rules.
DocstringParsingError: The NatSpec tags in the comment block cannot be parsed.
SyntaxError: Syntactical error, such as continue is used outside of a for loop.
DeclarationError: Invalid, unresolvable or clashing identifier names. e.g. Identifier not found
TypeError: Error within the type system, such as invalid type conversions, invalid assignments, etc.
UnimplementedFeatureError: Feature is not supported by the compiler, but is expected to be supported in future versions.
InternalCompilerError: Internal bug triggered in the compiler - this should be reported as an issue.
Exception: Unknown failure during compilation - this should be reported as an issue.
CompilerError: Invalid use of the compiler stack - this should be reported as an issue.
FatalError: Fatal error not processed correctly - this should be reported as an issue.
Warning: A warning, which didn’t stop the compilation, but should be addressed if possible.
© Copyright 2016-2019, Ethereum Revision 6da8b019.

Built with Sphinx using a theme provided by Read the Docs. to GitHub

Welcome to GitHub—where millions of developers work together on software. Ready to get started? Let’s learn how this all works by building and publishing your first GitHub Pages website!

## Repositories

Right now, we’re in your first GitHub **repository**. A repository is like a folder or storage space for your project. Your project's repository contains all its files such as code, documentation, images, and more. It also tracks every change that you—or your collaborators—make to each file, so you can always go back to previous versions of your project if you make any mistakes.

This repository contains three important files: The HTML code for your first website on GitHub, the CSS stylesheet that decorates your website with colors and fonts, and the **README** file. It also contains an image folder, with one image file.

## Describe your project

You are currently viewing your project's **README** file. **_README_** files are like cover pages or elevator pitches for your project. They are written in plain text or [Markdown language](https://guides.github.com/features/mastering-markdown/), and usually include a paragraph describing the project, directions on how to use it, who authored it, and more.

[Learn more about READMEs](https://help.github.com/en/articles/about-readmes)

## Your first website

**GitHub Pages** is a free and easy way to create a website using the code that lives in your GitHub repositories. You can use GitHub Pages to build a portfolio of your work, create a personal website, or share a fun project that you coded with the world. GitHub Pages is automatically enabled in this repository, but when you create new repositories in the future, the steps to launch a GitHub Pages website will be slightly different.

[Learn more about GitHub Pages](https://pages.github.com/)

## Rename this repository to publish your site

We've already set-up a GitHub Pages website for you, based on your personal username. This repository is called `hello-world`, but you'll rename it to: `username.github.io`, to match your website's URL address. If the first part of the repository doesn’t exactly match your username, it won’t work, so make sure to get it right.

Let's get started! To update this repository’s name, click the `Settings` tab on this page. This will take you to your repository’s settings page. 

![repo-settings-image](https://user-images.githubusercontent.com/18093541/63130482-99e6ad80-bf88-11e9-99a1-d3cf1660b47e.png)

Under the **Repository Name** heading, type: `username.github.io`, where username is your username on GitHub. Then click **Rename**—and that’s it. When you’re done, click your repository name or browser’s back button to return to this page.

<img width="1039" alt="rename_screenshot" src="https://user-images.githubusercontent.com/18093541/63129466-956cc580-bf85-11e9-92d8-b028dd483fa5.png">

Once you click **Rename**, your website will automatically be published at: https://your-username.github.io/. The HTML file—called `index.html`—is rendered as the home page and you'll be making changes to this file in the next step.

Congratulations! You just launched your first GitHub Pages website. It's now live to share with the entire world

## Making your first edit

When you make any change to any file in your project, you’re making a **commit**. If you fix a typo, update a filename, or edit your code, you can add it to GitHub as a commit. Your commits represent your project’s entire history—and they’re all saved in your project’s repository.

With each commit, you have the opportunity to write a **commit message**, a short, meaningful comment describing the change you’re making to a file. So you always know exactly what changed, no matter when you return to a commit.

## Practice: Customize your first GitHub website by writing HTML code

Want to edit the site you just published? Let’s practice commits by introducing yourself in your `index.html` file. Don’t worry about getting it right the first time—you can always build on your introduction later.

Let’s start with this template:

```
<p>Hello World! I’m [username]. This is my website!</p>
```

To add your introduction, copy our template and click the edit pencil icon at the top right hand corner of the `index.html` file.

<img width="997" alt="edit-this-file" src="https://user-images.githubusercontent.com/18093541/63131820-0794d880-bf8d-11e9-8b3d-c096355e9389.png">


Delete this placeholder line:

```
<p>Welcome to your first GitHub Pages website!</p>
```

Then, paste the template to line 15 and fill in the blanks.

<img width="1032" alt="edit-githuboctocat-index" src="https://user-images.githubusercontent.com/18093541/63132339-c3a2d300-bf8e-11e9-8222-59c2702f6c42.png">


When you’re done, scroll down to the `Commit changes` section near the bottom of the edit page. Add a short message explaining your change, like "Add my introduction", then click `Commit changes`.


<img width="1030" alt="add-my-username" src="https://user-images.githubusercontent.com/18093541/63131801-efbd5480-bf8c-11e9-9806-89273f027d16.png">

Once you click `Commit changes`, your changes will automatically be published on your GitHub Pages website. Refresh the page to see your new changes live in action.

:tada: You just made your first commit! :tada:

## Extra Credit: Keep on building!

Change the placeholder Octocat gif on your GitHub Pages website by [creating your own personal Octocat emoji](https://myoctocat.com/build-your-octocat/) or [choose a different Octocat gif from our logo library here](https://octodex.github.com/). Add that image to line 12 of your `index.html` file, in place of the `<img src=` link.

Want to add even more code and fun styles to your GitHub Pages website? [Follow these instructions](https://github.com/github/personal-website) to build a fully-fledged static website.

![octocat](./images/create-octocat.png)

## Everything you need to know about GitHub

Getting started is the hardest part. If there’s anything you’d like to know as you get started with GitHub, try searching [GitHub Help](https://help.github.com). Our documentation has tutorials on everything from changing your repository settings to configuring GitHub from your command line.
