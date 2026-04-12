The goal of these repositories is to implement filesystem shrinking for bcachefs.
They each use their own nix flakes to get the required dependencies, so make sure to use the correct ones. The kernels devshell is located elsewhere, but direnv still works.
When editing the bcachefs source, you do not have to update the userspace mirror in the bcachefs-tools repository.

Whenever you discover something that would be good to have in this AGENTS.md file, add it.

Always describe your changes and make them understandable to me after you're done.
Document anything tricky or noteworthy directly in the code. The end goal of these changes is to get merged into mainline, any aid in understandability is good.

## Commands
### general
Use jj for version control, commit whenever you find it appropriate, you can also rewrite the history of your _own_ commits if you find it approriate.
### bcachefs
check if it compiles - `make W=1 O=out -j (nproc) fs/bcachefs/`
### ketst
- run shrink tests
- after both of the commands below the test name(s) to run can be specified, if only some of them should be run. For this, strip the test_ prefix from the test name. (./build-test-kernel run ... online_two_device_data_shrink)
- remember to stop the process(es) after you've got the information you wanted, as -I (and -R) keeps them alive until explicitly stopped.
#### single-run
`./build-test-kernel run -I -K -k ../bcachefs tests/fs/bcachefs/shrink.ktest`
#### multi-run
Until now more useful as some of the tests are quite flaky, and might even have multiple different errors that might occur.
This command runs the test(s) in a loop until interrupted. Wait for a number of iterations that seems appropriate.
`./build-test-kernel run -R -I -K -k ../bcachefs tests/fs/bcachefs/shrink.ktest`
