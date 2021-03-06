# keytar

This is a fork of the [node-keytar](https://github.com/atom/node-keytar) package
that is designed to run as a standalone C++ library outside of a Node.js
environment.  

This package is designed to add, get, find, replace, and delete passwords in the
system's keychain.  On OS X the passwords are managed by the Keychain, on Linux
they are managed by GNOME Keyring, and on Windows they are managed by Credential
Vault.


## Status

The module is currently tested<sup>1</sup> on the following platforms:

| Windows                           | OS X/Linux                             |
| :-------------------------------: | :------------------------------------: |
| [![Windows][win-badge]][win-link] | [![OS X][osx-lin-badge]][osx-lin-link] |

[win-badge]: https://ci.appveyor.com/api/projects/status/8ck7jhmx0p2d8qcn/branch/master?svg=true "AppVeyor build status"
[win-link]:  https://ci.appveyor.com/project/havoc-io/keytar/branch/master "AppVeyor build status"
[osx-lin-badge]: https://travis-ci.org/havoc-io/keytar.svg?branch=master "Travis CI build status"
[osx-lin-link]:  https://travis-ci.org/havoc-io/keytar "Travis CI build status"

<sup>
1: Sadly, the gnome-keyring-daemon does not work on Travis CI, so while the
library and tests are built on Linux, the tests are not actually run.  If you
want to execute the tests, you'll have to build and run them locally :cry:.
You'll probably have a lot better luck if you do this in a GNOME session.
</sup>


## Dependencies

On Windows and OS X, all dependencies are met by the system.

On Linux, you need to ensure that the GNOME Keyring development package is
installed.  On Ubuntu systems, do:

    sudo apt-get install libgnome-keyring-dev

On Red Hat systems, do:

    sudo yum install gnome-keyring-devel

For all other Linux systems, consult your package manager.


## Building

The build system is CMake-based, with standard behavior.  To build, simply do
something like:

    mkdir build
    cd build
    cmake ..
    make

Adjust for your particular platform, generator, and desired settings.  The build
system supports the following options:

- **BUILD_SHARED_LIBS**: Whether to build libuv as a shared library (`ON` by
  default) or a static library (`OFF`)
- **BUILD_TESTS**: Whether or not to build libuv tests (`ON` by default)
- **BIN_INSTALL_DIR**: The subdirectory of `CMAKE_INSTALL_PREFIX` which will be
  used to install runtime binaries (`bin` by default) (note that, on Windows,
  CMake considers `.dll` files to be runtime binaries, and will stick them in
  the binary install directory, while the corresponding export `.lib` files will
  be stuck in the library install directory)
- **LIB_INSTALL_DIR**: The subdirectory of `CMAKE_INSTALL_PREFIX` which will be
  used to install libraries (`lib` by default)
- **INCLUDE_INSTALL_DIR**: The subdirectory of `CMAKE_INSTALL_PREFIX` which will
  be used to install include headers (`include` by default)

These options can be controlled by CMake's `-D` command line flags or by using
`ccmake` to edit the cache.

An `install` target is also generated that follows standard CMake conventions.


## Usage

The API is exposed via the header file `keytar.h`.  The "replace" functionality
implemented in node-keytar is simply a convenience wrapper, and is not
replicated here.

Example usage:

    // Standard includes
    #include <string>
    #include <iostream>

    // keytar includes
    #include <keytar.h>

    int main() {
        // Test values
        const std::string service("keytar-test-service");
        const std::string account("keytar@example.org");
        const std::string password("$uP3RseCr1t!");
        const std::string new_password("Ub3R$3CrE7!?!");

        // Add a password
        if (!keytar::AddPassword(service, account, password)) {
            std::cerr << "error: unable to add password" << std::endl;
            return 1;
        }

        // Get a password
        std::string retrieved;
        if (!keytar::GetPassword(service, account, &retrieved)) {
            std::cerr << "error: unable to get password" << std::endl;
            return 1;
        }

        // Find a password
        std::string found;
        if (!keytar::FindPassword(service, &found)) {
            std::cerr << "error: unable to find password" << std::endl;
            return 1;
        }

        // Replace a password
        if (!keytar::ReplacePassword(service, account, new_password)) {
            std::cerr << "error: unable to replace password" << std::endl;
            return 1;
        }

        // Delete a password
        if (!keytar::DeletePassword(service, account)) {
            std::cerr << "error: unable to delete password" << std::endl;
            return 1;
        }

        // All done
        return 0;
    }


## Updating

This fork will attempt to keep in sync with node-keytar, and tags will be
duplicated but suffixed with `-standalone`.  The code remains largely
unmodified, but some modifications were necessary to make shared libraries work.

The strategy for bringing in upstream changes is as follows:

1. First, make sure that the node-keytar repository is available as a remote for
   your copy of keytar.  You can check this by running `git remote -v` and
   seeing if a remote named `upstream` with the correct location appears in the
   output.  If not, you just need to run
   `git remote add upstream https://github.com/atom/node-keytar.git`.  You can,
   of course, use a different remote name than `upstream`, but the rest of these
   instructions will operate under the assumption that this is the name you have
   chosen.
2. Fetch a copy of the upstream changes with `git fetch upstream` and the
   upstream tags with `git fetch --tags upstream`.
3. Merge in upstream changes by switching to keytar's `master` branch and
   running `git merge upstream/master`.
3. It's very likely that you'll have conflicts, which will generally be of the
   following types:
   - Upstream modifications to files that don't exist in keytar because they
     have been deleted.  If these files are not necessary, simply `git rm` them
     from the conflicted merge state.
   - Upstream modifications to files that do exist in keytar but have been
     changed by both projects.  In this case you have to handle the conflicts
     like any other, and then `git add` them.
4. Once you've sorted out any merge conflicts, simply run `git commit` and then
   push.  If, by some miracle, you didn't have any merge conflicts, then
   congratulations, you can just continue to the next step without any conflict
   resolution.
5. Push the merged changes with `git push` and push upstream tags with
   `git push --tags`.
6. Make sure CI tests pass.
7. If any node-keytar tags have been added, these should ideally be mirrored in
   keytar with the `-standalone` suffix.  Depending on how lazy the maintainer
   is, multiple tags may have been added in between upstream merges, but unless
   you do a merge at each upstream tag, you'll only be able to create keytar
   tags corresponding to the latest upstream tag, because the intermediate tags
   won't have any corresponding merged commit in keytar.  As long as we keep up
   with things, this shouldn't be a problem.  Even if we skip one or two
   upstream releases, no biggie.
