.. _custom_chisel:

Integrating Custom Chisel Projects into the Generator Build System
==================================================================

.. warning::
   This section assumes integration of custom Chisel through git submodules.
   While it is possible to directly commit custom Chisel into the Chipyard framework,
   we heavily recommend managing custom code through git submodules. Using submodules decouples
   development of custom features from development on the Chipyard framework.


While developing, you want to include Chisel code in a submodule so that it can be shared by different projects.
To add a submodule to the Chipyard framework, make sure that your project is organized as follows.

.. code-block:: none

    yourproject/
        build.sbt
        src/main/scala/
            YourFile.scala

Put this in a git repository and make it accessible.
Then add it as a submodule to under the following directory hierarchy: ``generators/yourproject``.

The ``build.sbt`` is a minimal file which describes metadata for a Chisel project.
For a simple project, the ``build.sbt`` can even be empty, but below we provide an example
build.sbt.

.. code-block:: scala

    organization := "edu.berkeley.cs"

    version := "1.0"

    name := "yourproject"

    scalaVersion := "2.12.4"



.. code-block:: shell

    cd generators/
    git submodule add https://git-repository.com/yourproject.git

Then add ``yourproject`` to the Chipyard top-level build.sbt file.

.. code-block:: scala

    lazy val yourproject = (project in file("generators/yourproject")).settings(commonSettings).dependsOn(rocketchip)

You can then import the classes defined in the submodule in a new project if
you add it as a dependency. For instance, if you want to use this code in
the ``chipyard`` project, add your project to the list of sub-projects in the
`.dependsOn()` for `lazy val chipyard`. The original code may change over time, but it
should look something like this:

.. code-block:: scala

    lazy val chipyard = (project in file("generators/chipyard"))
        .dependsOn(testchipip, rocketchip, boom, hwacha, rocketchip_blocks, rocketchip_inclusive_cache, iocell,
            sha3, dsptools, `rocket-dsp-utils`,
            gemmini, icenet, tracegen, cva6, nvdla, sodor, ibex, fft_generator,
            yourproject, // <- added to the middle of the list for simplicity
            constellation, mempress)
        .settings(libraryDependencies ++= rocketLibDeps.value)
        .settings(
            libraryDependencies ++= Seq(
            "org.reflections" % "reflections" % "0.10.2"
            )
        )
        .settings(commonSettings)

We also noticed the need to verify Chisel modules standalone without Chipyard. This could get tricky, especially when other Chisel libraries, such as Berkeley hardfloat, are involved. In this case, you could setup the root folder of your accelerator like this:

.. code-block:: none

    yourproject/
        dependencies/hardfloat/...
        project/
            plugins.sbt
            build.properties
        build.sbt
        src/main/scala/
            YourFile.scala

Please make sure your external library, in this example hardfloat, is cloned (or git submodules initialized) to the ``dependencies`` directory.

The content of ``project/plugins.sbt`` should be like:

.. code-block:: scala

   addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.1.1")
   addSbtPlugin("ch.epfl.scala" % "sbt-scalafix" % "0.10.4")
   addSbtPlugin("ch.epfl.scala" % "sbt-bloop" % "1.5.6")

The content of ``project/build.properties`` should be like:

.. code-block:: scala

   sbt.version = 1.8.2

Then you could mimic the following ``build.sbt`` to set up the environment at the root folder of your accelerator:

.. code-block:: scala

   lazy val commonSettings = Seq(
     scalaVersion := "2.13.10",
     scalacOptions ++= Seq(
       "-deprecation",
       "-unchecked",
       "-Ymacro-annotations"), // fix hierarchy API
     )

   /**
     * It has been a struggle for us to override settings in subprojects.
     * An example would be adding a dependency to rocketchip on midas's targetutils library,
     * or replacing dsptools's maven dependency on chisel with the local chisel project.
     *
     * This function works around this by specifying the project's root at src/ and overriding
     * scalaSource and resourceDirectory.
     */
   def freshProject(name: String, dir: File): Project = {
     Project(id = name, base = dir / "src")
       .settings(
         Compile / scalaSource := baseDirectory.value / "main" / "scala",
         Compile / resourceDirectory := baseDirectory.value / "main" / "resources"
       )
   }
   
   val chiselVersion = "3.6.0"
   
   lazy val chiselSettings = Seq(
     libraryDependencies ++= Seq("edu.berkeley.cs" %% "chisel3" % chiselVersion,
     "org.apache.commons" % "commons-lang3" % "3.12.0",
     "org.apache.commons" % "commons-text" % "1.9"),
     addCompilerPlugin("edu.berkeley.cs" % "chisel3-plugin" % chiselVersion cross CrossVersion.full))
   
   
   lazy val hardfloat = freshProject("hardfloat", file("dependencies/hardfloat/hardfloat"))
     .settings(chiselSettings)
     .settings(commonSettings)
   
   
   lazy val root = (project in file("."))
       .dependsOn(hardfloat)
       .settings(
           name := "yourproject",
           version := "0.1.0",
         )
       .settings(chiselSettings)
       .settings(commonSettings)
       .settings(
           libraryDependencies += "edu.berkeley.cs" %% "chiseltest" % "0.6.0" % Test,
       )

Note that all the above version numbers may change as the resources upgrade, but the general structure should serve as a good guideline.

Also, it is fine to keep this build file at the root of your accelerator repo while integrating into chipyard, as long as your accelerator is declared with the ``freshProject`` call at the top-level ``build.sbt``. For example:

.. code-block:: scala

   lazy val hardfloat = freshProject("yourproject", file("generators/yourproject"))
     .settings(chiselSettings)
     .settings(commonSettings)
