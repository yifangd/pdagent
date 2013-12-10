#
# Build script for agent.
#

import os
import subprocess
import sys


def cleanup(target, source, env):
    """Removes generated artifacts"""
    # TODO clean up.
    pass


def createDist(target, source, env):
    """Create distributable for agent."""
    # TODO copy packages, documentation?
    pass


def createPackages(target, source, env):
    """Create installable packages for supported operating systems."""
    retCode = 0
    retCode += _createDebPackage()
    retCode += _createRpmPackage()
    return retCode


def runIntegrationTests(target, source, env):
    """Run integration tests on running virts."""
    source_paths = [s.path for s in source]
    test_runner_file = _generate_remote_test_runner_file(
        source_paths,
        lambda f: f.startswith("test_") and f.endswith(".sh"),
        executable="sh")
    return _run_on_virts("sh %s" % test_runner_file)


def runUnitTests(target, source, env):
    """Run unit tests on running virts."""
    source_paths = [s.path for s in source]
    test_runner_file = _generate_remote_test_runner_file(
        source_paths,
        lambda f: f.startswith("test_") and f.endswith(".py"))
    return _run_on_virts("%s %s" % (sys.executable, test_runner_file))


def runUnitTestsLocal(target, source, env):
    """Run unit tests on current machine."""
    source_paths = [s.path for s in source]
    test_files = _getFilePathsRecursive(
        source_paths,
        lambda f: f.startswith("test_") and f.endswith(".py"))
    test_files.sort()

    total = 0
    errs = 0
    test_env = os.environ.copy()
    test_env["PYTHONPATH"] = \
        test_env.get("PYTHONPATH", "") + os.pathsep + env.Dir(".").abspath
    for test_file in test_files:
        print "FILE: %s" % test_file
        exit_code = subprocess.call([sys.executable, test_file], env=test_env)
        total += 1
        errs += (exit_code != 0)
    print "SUMMARY: %s total / %s error (%s)" % (total, errs, sys.executable)
    return errs


def startVirtualBoxes(target, source, env):
    virts = env.get("virts", [])
    start_cmd = ["vagrant", "up"]
    start_cmd.extend(virts)
    return subprocess.call(start_cmd)


def _createDebPackage():
    # TODO create the package.
    print "\nCreating .deb package..."
    return 0


def _createRpmPackage():
    # TODO create the package.
    print "\nCreating .rpm package..."
    return 0


def _generate_remote_test_runner_file(
    source_paths,
    test_filename_matcher,
    executable=sys.executable):

    test_dir = os.path.join("target", "tmp")
    env.Execute(Mkdir(test_dir))
    test_runner_file = os.path.join(test_dir, "run_tests")

    test_files = _getFilePathsRecursive(source_paths, test_filename_matcher)
    test_run_paths = [os.path.join(os.sep, "vagrant", t) for t in test_files]

    run_commands = ["e=0"]
    for test in test_run_paths:
        run_commands.append(" ".join([executable, test]))
        run_commands.append("e=$(( $e + $? ))")
    run_commands.append("exit $e")

    #TODO this doesn't work -- 'Textfile' is not recognized.
#     env.Textfile(
#         target=test_runner_file,
#         source=run_commands)
    out = open(test_runner_file, "w")
    out.write(os.linesep.join(run_commands))
    out.flush()
    out.close()

    return os.path.join(os.sep, "vagrant", test_runner_file)


def _getFilePathsRecursive(source_paths, filename_matcher):
    dirs_traversed = set()
    files = set()

    def _addFiles(dir_path):
        dirs_traversed.add(dir_path)
        for dirname, subdirnames, filenames in os.walk(dir_path):
            for subdir in subdirnames:
                dirs_traversed.add(os.path.join(dirname, subdir))
            for filename in filenames:
                if filename_matcher(filename):
                    files.add(os.path.join(dirname, filename))

    for src in source_paths:
        if os.path.isdir(src):
            if not src in dirs_traversed:
                _addFiles(src)
        else:
            if filename_matcher(os.path.basename(src)):
                files.add(src)
    return list(files)


def _get_arg_values(key, default=None):
    values = [v for k, v in ARGLIST if k == key]
    if not values and default:
        values = default
    return values


def _get_virt_names():
    return [v.split()[0] for v in \
        subprocess \
        .check_output(["vagrant", "status"]) \
        .splitlines() \
        if v.find("running") >= 0]


def _run_on_virts(remote_command):
    exit_code = 0
    for virt in _get_virt_names():
        command = ["vagrant", "ssh", virt, "-c", remote_command]
        print "Running on %s..." % virt
        exit_code += subprocess.call(command)
    return exit_code


env = Environment()
env.Alias("all", ["."])

# TODO update help when commands are finalized.
env.Help("""
Usage: scons [command [command...]]
where supported commands are:
all                 Runs all commands.
clean               Removes generated artifacts.
dist                Creates distributable artifacts for agent.
package             Creates installable packages for supported OS
                    distributions.
                    This is the default command if none is specified.
test                Runs unit tests.
                    By default, runs all tests in `pdagenttest` recursively.
                    (Test files should be named in the format `test_*.py`.)
                    Specific unit tests can be run by providing them as
                    arguments to this option, multiple times if required.
                    Both test files and test directories are supported.
                    e.g.
                    scons test=pdagenttest/test_foo.py test=pdagenttest/queue
test-integration    Runs integration tests.
""")

unitTestLocalTask = env.Command(
    "test-local",
    _get_arg_values("test-local", ["pdagenttest"]),
    env.Action(runUnitTestsLocal, "\n--- Running unit tests locally"))

startVirtsTask = env.Command(
    "start-virt",
    None,
    env.Action(startVirtualBoxes, "\n--- Starting virtual boxes"),
    virts=_get_arg_values("start-virt"))

unitTestTask = env.Command(
    "test",
    _get_arg_values("test", ["pdagenttest"]),
    env.Action(runUnitTests,
        "\n--- Running unit tests on virtual boxes"))
env.Requires(unitTestTask, startVirtsTask)

createPackagesTask = env.Command(
    "package",
    None,
    env.Action(createPackages, "\n--- Creating install packages"))
env.Requires(createPackagesTask, unitTestTask)

integrationTestTask = env.Command(
    "test-integration",
    _get_arg_values("test-integration", ["pdagenttestinteg"]),
    env.Action(runIntegrationTests,
        "\n--- Running integration tests on virtual boxes"))
env.Requires(integrationTestTask, [createPackagesTask, startVirtsTask])

distTask = env.Command(
    "dist",
    None,
    env.Action(createDist, "\n--- Creating distributables"))
env.Requires(distTask, [unitTestTask, createPackagesTask, integrationTestTask])

cleanTask = env.Command(
    "clean",
    None,
    env.Action(cleanup, "\n--- Cleaning up"))

# task to run if no command is specified.
env.Default(createPackagesTask)
