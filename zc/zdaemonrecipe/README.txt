zdaemon recipe
==============

zc.zdaemonrecipe provides a recipe that can be used to create zdaemon
run-control scripts. It can be used directly in buildouts and by other
recipes.

It accepts 2 options:

program
   The anme of the program and, optionally, command-line arguments.
   (Note that, due to limitations in zdaemon, the command-line options
   cannot have embedded spaces.)

zdaemon.conf
   Optionally, you can provide extra configuration in ZConfig format.
   What's provided will be augmented by the zdaemon recipe, as needed.

deployment
   The name of a zc.recipe.deployment deployment.  If specified, then:

   - The configuration, log, and run-time files will be put in
     deployment-defined directories.

   - A logrotate configuration will be generated for the zdaemon
     transacript log.

Let's look at an example:

    >>> write('buildout.cfg',
    ... '''
    ... [buildout]
    ... parts = run
    ...
    ... [run]
    ... recipe = zc.zdaemonrecipe
    ... program = sleep 1
    ... ''')

If we run the buildout, we'll get a run part that contains the zdaemon
configuration file and a run script in the bin directory:

    >>> print system(buildout),
    Installing run.
    Generated script '/sample-buildout/bin/zdaemon'.
    Generated script '/sample-buildout/bin/run'.

    >>> cat('parts', 'run', 'zdaemon.conf')
    <runner>
      daemon on
      directory /sample-buildout/parts/run
      program sleep 1
      socket-name /sample-buildout/parts/run/zdaemon.sock
      transcript /sample-buildout/parts/run/transcript.log
    </runner>
    <BLANKLINE>
    <eventlog>
      <logfile>
        path /sample-buildout/parts/run/transcript.log
      </logfile>
    </eventlog>

    >>> cat('bin', 'run') # doctest: +NORMALIZE_WHITESPACE
    #!/usr/local/bin/python2.4
    <BLANKLINE>
    import sys
    sys.path[0:0] = [
      '/sample-buildout/eggs/zdaemon-2.0-py2.4.egg',
      '/sample-buildout/eggs/ZConfig-2.4-py2.4.egg',
      ]
    <BLANKLINE>
    import zdaemon.zdctl
    <BLANKLINE>
    if __name__ == '__main__':
        sys.exit(zdaemon.zdctl.main([
            '-C', '/sample-buildout/parts/run/zdaemon.conf',
            ]+sys.argv[1:]
            ))

zdaemon will also be installed:

    >>> ls('eggs')
    d  ZConfig-2.4-py2.4.egg
    d  setuptools-0.6-py2.4.egg
    d  zc.buildout-1.0.0b27-py2.4.egg
    d  zc.recipe.egg-1.0.0-py2.4.egg
    d  zdaemon-2.0-py2.4.egg
    d  zope.testing-3.4-py2.4.egg

You can use an eggs option to specify a zdaemon version.


If we specify a deployment, then the files will be placed in
deployment-defined locations:

    >>> mkdir('etc')
    >>> mkdir('init.d')
    >>> mkdir('logrotate')

    >>> write('buildout.cfg',
    ... '''
    ... [buildout]
    ... parts = run
    ...
    ... [run]
    ... recipe = zc.zdaemonrecipe
    ... program = sleep 1
    ... deployment = deploy
    ...
    ... [deploy]
    ... name = test-deploy
    ... etc-directory = etc
    ... rc-directory = init.d
    ... log-directory = logs
    ... run-directory = run
    ... logrotate-directory = logrotate
    ... user = bob
    ... ''')

    >>> print system(buildout),
    Uninstalling run.
    Installing run.
    Generated script '/sample-buildout/init.d/test-deploy-run'.

    >>> import os
    >>> os.path.exists('parts/run')
    False

    >>> os.path.exists('bin/run')
    False

    >>> cat('etc', 'run-zdaemon.conf')
    <runner>
      daemon on
      directory run
      program sleep 1
      socket-name run/run-zdaemon.sock
      transcript logs/run.log
      user bob
    </runner>
    <BLANKLINE>
    <eventlog>
      <logfile>
        path logs/run.log
      </logfile>
    </eventlog>

    >>> cat('init.d', 'test-deploy-run') # doctest: +NORMALIZE_WHITESPACE
    #!/usr/local/bin/python2.4
    <BLANKLINE>
    import sys
    sys.path[0:0] = [
      '/sample-buildout/eggs/zdaemon-2.0a6-py2.4.egg',
      '/sample-buildout/eggs/ZConfig-2.4a6-py2.4.egg',
      ]
    <BLANKLINE>
    import zdaemon.zdctl
    <BLANKLINE>
    if __name__ == '__main__':
        sys.exit(zdaemon.zdctl.main([
            '-C', 'etc/run-zdaemon.conf',
            ]+sys.argv[1:]
            ))

    >>> cat('logrotate', 'test-deploy-run')
    logs/run.log {
      rotate 5
      weekly
      postrotate
        init.d/test-deploy-run -C etc/run-zdaemon.conf reopen_transcript
      endscript
    }

If you want to override any part of the generated zdaemon configuration,
simply provide a zdaemon.conf option in your instance section:

    >>> write('buildout.cfg',
    ... '''
    ... [buildout]
    ... parts = run
    ...
    ... [run]
    ... recipe = zc.zdaemonrecipe
    ... program = sleep 1
    ... deployment = deploy
    ... zdaemon.conf =
    ...     <runner>
    ...       daemon off
    ...       socket-name /sample-buildout/parts/instance/sock
    ...       transcript /dev/null
    ...     </runner>
    ...     <eventlog>
    ...     </eventlog>
    ...
    ... [deploy]
    ... etc-directory = etc
    ... rc-directory = init.d
    ... log-directory = logs
    ... run-directory = run
    ... logrotate-directory = logrotate
    ... user = bob
    ... ''')

    >>> print system(buildout),
    Uninstalling run.
    Installing run.
    Generated script '/sample-buildout/init.d/deploy-run'.

    >>> cat('etc', 'run-zdaemon.conf')
    <runner>
      daemon off
      directory run
      program sleep 1
      socket-name /sample-buildout/parts/instance/sock
      transcript /dev/null
      user bob
    </runner>
    <BLANKLINE>
    <eventlog>
    </eventlog>


Creating shell start scripts
----------------------------

By default, the startup scripts are generated Python scripts that use
the zdaemon module.  Sometimes, this is inconvenient.  In particular,
when deploying software, generated Python scripts may break after a
software update because they contain pasths to software eggs.  We can
request shell scripts that invoke a generic zdaemon script.  The shell
script only depends on the path to the zdaemon script, which generally
doesn't change when updating softawre.

To request a shell script, add a shell-script option with a true value:

    >>> write('buildout.cfg',
    ... '''
    ... [buildout]
    ... parts = run
    ...
    ... [run]
    ... recipe = zc.zdaemonrecipe
    ... program = sleep 1
    ... shell-script = true
    ... deployment = deploy
    ...
    ... [deploy]
    ... name = test-deploy
    ... etc-directory = etc
    ... rc-directory = init.d
    ... log-directory = logs
    ... run-directory = run
    ... logrotate-directory = logrotate
    ... user = alice
    ... ''')

    >>> print system(buildout),  # doctest: +NORMALIZE_WHITESPACE
    Uninstalling run.
    Installing run.
    zc.zdaemonrecipe: Generated shell script
        '/sample-buildout/init.d/test-deploy-run'.

    >>> cat('init.d', 'test-deploy-run')  # doctest: +NORMALIZE_WHITESPACE
    #!/bin/sh
    su alice -c \
        "/sample-buildout/bin/zdaemon
             -C '/sample-buildout/etc/run-zdaemon.conf' $*"

    >>> ls('etc')
    -  run-zdaemon.conf


Using the zdaemon recipe from other recipes
-------------------------------------------

To use the daemon recipe from other recipes, simply instantiate an
instance in your recipe __init__, passing your __init__ arguments, and
then calling the instance's install in your install method.
