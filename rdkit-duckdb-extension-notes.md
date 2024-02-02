- Wasn't sure how to bring rdkit into duckdb. Looked at chemicalite to see how they did it for SQLite.
  Thought this could be helpful because chemicalite is not part of the RDKit code base, so maybe it would
  have some similarity to what I would like to do with duckdb.

- `CMakeLists.txt` in the root dir has some hints as well as the documentation.
  Requires RDKit to be installed

```Cmake
find_package(RDKit 2020.09.1 REQUIRED)
include_directories(${RDKit_INCLUDE_DIRS})
```

- `src/CMakeLists.txt` has a function `add_library` seems to add the cpp files,
  such as `mol.cpp` in

Then this seems like this brings in the RDKit modules?
I think this sets the variable `CHEMICALITE_RDKIT_LIBRARIES` with the following values

```Cmake
set(CHEMICALITE_RDKIT_LIBRARIES
    RDKit::MolHash
    RDKit::Descriptors
    RDKit::Fingerprints
    RDKit::FileParsers
    RDKit::ChemTransforms
    RDKit::FMCS
    RDKit::MolStandardize
    RDKit::GraphMol
    )
```

- What are the minimal and basic functions needed?

  - looked at minimal lib which has functions for rdkitJS: https://github.com/rdkit/rdkit/blob/7ba3be0597133ca6a40ec66193984aca555e8a80/Code/MinimalLib/minilib.h

- try get a dev environment set up with the extension template that has rdkit loaded in
- start with mol_from_smiles `mol_formats.cpp` line 102
  - this seems to do some checks for SQLite, and then uses `RDKit::SmilesToMol(smiles)` to make a mol object (sets a pointer to that object)

### Getting duckdb with extension template setup

- followed instructions, removed anything with the vcpkg and OpenSSL example of dependencies
- needed to change in CMakeLists.txt:

```cmake
cmake_minimum_required(VERSION 3.5)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

this generates a compile_commands.json in build/release which tells clangd, the LSP in neovim, where files are. Otherwise, it says it can't find anything
Make a symbolic link to the root dir which is where clangd looks for this file
`ln -s build/release/compile_commands.json .`

compiles and run the shell with `./build/release/duckdb` and it works with the example extension, quack

### Bringing RDKit into duckdb code

- tried building rdkit myself and it wasn't working. problems with python and boost
- abandoned that and followed what was recommended in rdkit docs, to use conda/mamba
- needed to install conda and mamba. Did not want to do this because I found conda a bit invasive in the past, but maybe I did things incorrectly.
- chose to use conda in my bash shell so that I don't have to deal with it in zsh which I typically use
- made an alias for mamba in zsh, then call that mamba init which will edit bashrc by default
- then switching to bash in zsh with just `bash` will have the base env activated
- made a rdkit_dev env according to: https://greglandrum.github.io/rdkit-blog/posts/2021-07-24-setting-up-a-cxx-dev-env.html
- added `find_package(RDKit REQUIRED)` to CMake
- went to the duckdb rdkit extension repo and tried make. That seemed to work

- needed to make a bit of changes to CMakeLists
  - followed example from : https://github.com/greglandrum/rdkit_blog/blob/master/src/simple_cxx_example/CMakeLists.txt
  - and also the duckdb extension template
  - also followed the chemicalite to figure out where certain functions may be in RDKit
  - was able to compile and run: `select duckdb_rdkit('hi') as result;`

### Looking into duckdb internals

- Started looking into the duckdb code because I didn't know how to write data to tables or disk. Or how to specify new types. I didn't know where to even begin
- Looked at spatial extension They seem to have new types there
- cloned the duckdb repo. Need to run `make clang` to build the compilation database and have the clangd LSP pickup header files

  - followed github.com/clangd/clangd/issues/1204

- looked at spatial:

  - found it uses LogicalType and something called ExtensionUtil::RegisterType to create the new types
  - this is in `spatial/src/spatial/core/types.cpp`
  - looked for this in duckdb
  - LogicalType in `duckdb/src/include/duckdb/common/types.hpp` defines SQL Types like FLOAT, DOUBLE, etc.

- looked for the ExtensionUtil in duckdb and found `src/main/extension/extension_util.cpp` and
  the header file `duckdb/src/include/duckdb/main/extension_util.hpp`
  There is something called RegisterType
- also `duckdb/src/common/types.cpp` shows how they made the internal duckdb types

- perhaps this can be used to create a molecule type from RDKit
- Left off at types.cpp in the duckdb_rdkit extension repo.
  Need to figure out what kind of LogicalType Mol should be. The Point2D example is a Struct of Logical Types with x and y for the coordinates.
  Check chemicalite to see what a mol is

### Looking at RDKit internals

- took a look at chemicalite and how they translate a molecule object into something SQLite can use.
  in mol_from_smiles (https://github.com/rvianello/chemicalite/blob/314fa98d9f3a233df88ba8d58a2bacd8afada8c9/src/mol_formats.cpp#L102), they change it into a blob

  ```c++
  Blob blob = mol_to_blob(\*mol, &rc);
  ```

- mol_to_blob uses mol_to_binary_mol ( std::string bmol = mol_to_binary_mol(mol, rc);)
- mol_to_binary_mol uses an RDKit function, Mol Pickler

```c++
{
  std::string buf;
  try {
    RDKit::MolPickler::pickleMol(mol, buf, RDKit::PicklerOps::AllProps);
  }
  catch (...) {
    *rc = SQLITE_ERROR;
    chemicalite_log(*rc, "Could not serialize mol to binary");
  }
  return buf;
}

```

- This is interesting! MolPickler serializes and deserializes the mol object (https://github.com/rdkit/rdkit/blob/Release_2023_09/Code/GraphMol/MolPickler.h)
  I want to now take a look at what is the molecule format.

- Here is information on the MolPickler::pickleMol function: https://github.com/rdkit/rdkit/blob/e4e2fe377fc44663c77825d483bc741328725169/Code/GraphMol/MolPickler.cpp#L896
- in the header file, we can find private functions that are used to handle atoms, bonds, other things about the molecule: https://github.com/rdkit/rdkit/blob/e4e2fe377fc44663c77825d483bc741328725169/Code/GraphMol/MolPickler.h#L206
- Here is how the function pickles atom data in a private function: https://github.com/rdkit/rdkit/blob/efd792237206a4305b70de2c76b3e3274403effb/Code/GraphMol/MolPickler.cpp#L1478
  It gets formal charge, chiral tags, etc.
- It seems to handle atom at a time, and writes different information about the atom? Yes, I think so. Looking at how you depickle an atom (https://github.com/rdkit/rdkit/blob/efd792237206a4305b70de2c76b3e3274403effb/Code/GraphMol/MolPickler.cpp#L1532)
  it seems to read the stream and do different things depending on the bytes
- Here is how it depickles the whole molecule and goes through each atom I think: https://github.com/rdkit/rdkit/blob/efd792237206a4305b70de2c76b3e3274403effb/Code/GraphMol/MolPickler.cpp#L1219

- also information here on Pickling molecules: https://www.rdkit.org/docs/GettingStartedInC%2B%2B.html#preserving-molecules
