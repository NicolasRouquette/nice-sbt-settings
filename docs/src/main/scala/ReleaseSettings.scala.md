## Release process

This module defines some new release steps and defines
a configurable sequence of the release process


```scala
package ohnosequences.sbt.nice

import sbt._
import Keys._
import sbt.Extracted._

import sbtrelease._, ReleasePlugin.autoImport._, ReleaseKeys._, ReleaseStateTransformations._

import DocumentationSettings._

import ohnosequences.sbt.SbtGithubReleasePlugin._
import ohnosequences.sbt.SbtGithubReleasePlugin.autoImport._

import laughedelic.literator.plugin.LiteratorPlugin.autoImport._

import com.markatta.sbttaglist._
import TagListPlugin._

import com.timushev.sbt.updates.UpdatesKeys


case object ReleaseSettings extends sbt.AutoPlugin {

  override def trigger = allRequirements
  override def requires =
    ohnosequences.sbt.nice.DocumentationSettings &&
    ohnosequences.sbt.nice.ResolverSettings &&
    ohnosequences.sbt.nice.ScalaSettings &&
    ohnosequences.sbt.nice.TagListSettings &&
    ohnosequences.sbt.nice.WartRemoverSettings &&
    ohnosequences.sbt.SbtGithubReleasePlugin &&
    sbtrelease.ReleasePlugin

  case object autoImport {

    lazy val releaseStepByStep = settingKey[Boolean]("Defines whether release process will wait for confirmation after each step")
    lazy val releaseOnlyTestTag = settingKey[String]("Sets the name for the tag that is used to distinguish release-only tests")
    lazy val releaseOnlyTestTagPackage = settingKey[String]("The package for release-only tags")
    lazy val testAll = taskKey[Unit]("Runs testOnly without args (runs all tests)")
  }
  import autoImport._
  import ReleaseSteps._
```

### Settings

```scala
  override lazy val projectSettings: Seq[Setting[_]] = Seq(
    libraryDependencies += "org.scalatest" %% "scalatest" % "2.2.6",

    releaseOnlyTestTagPackage := s"${organization.value}.test.tags",
    releaseOnlyTestTag := "ReleaseOnlyTest",

    sourceGenerators in Test += generateTestTags.taskValue,
```

Release only test tag is excluded by default

```scala
    testOptions in (Test, test) += Tests.Argument("-l", s"${releaseOnlyTestTagPackage}.${releaseOnlyTestTag.value}"),
    testAll := (testOnly in Test).toTask("").value,
```

We want to increment `y` in `x.y.z`

```scala
    releaseVersionBump := Version.Bump.Minor,
```

By default you want to have full controll over the release process:

```scala
    releaseStepByStep := true,

    releaseTagComment := {organization.value +"/"+ name.value +" v"+ (version in ThisBuild).value},
```

Adding release notes to the commit message

```scala
    releaseCommitMessage := {
      val log = streams.value.log
      val v = (version in ThisBuild).value
      val note: File = baseDirectory.value / "notes" / (v+".markdown")
      val text: String = IO.read(note)
      "Setting version to " +v+ ":\n\n"+ text
    },
```

This is a sequence of blocks (see them below)

```scala
    releaseProcess := {
      import ReleaseBlocks._

      constructReleaseProcess(
        initChecks,
        Seq(
          packAndTest,
          askVersionsAndCheckNotes,
          genMdDocs,
          genApiDocs,
          publishArtifacts,
          commitAndTag,
          githubRelease,
          nextVersion,
          githubPush
        )
      )
    }
  )
}

case object ReleaseBlocks {
  import ReleaseSteps._
  import ReleaseSettings.autoImport._
```

#### Initial checks

- check that release doesn't have snapshot or outdated dependencies
- check that we can use Github api for publishing notes
- warn if we have `TODO` or `FIXME` notes


```scala
  val initChecks = ReleaseBlock("Initial checks", Seq(
    checkNoSnapshotDeps,
    checkDependecyUpdates,
    checkTagList,
    releaseTask(checkGithubCredentials)
  ), transit = true)
```

#### Packaging and running tests

- try to pack
- run tests (including release-only)


```scala
  val packAndTest = ReleaseBlock("Packaging and running tests", Seq(
    releaseTask(Keys.`package`),
    releaseTask(testAll)
  ), transit = true)
```

#### Setting release version

- inquire the current and the next release versions
- set the current one (no commiting)
- check and confirm release notes for this version


```scala
  val askVersionsAndCheckNotes = ReleaseBlock("Setting release version", Seq(
    inquireVersions.action,
    tempSetVersion,
    checkReleaseNotes
  ), transit = true)
```

#### Generating markdown documentation

```scala
  val genMdDocs = ReleaseBlock("Generating markdown documentation", Seq(cleanAndGenerateDocsAction))
```

#### Generating api documentation and pushing to gh-pages

```scala
  val genApiDocs = ReleaseBlock("Generating api documentation and pushing to gh-pages", Seq(pushApiDocsToGHPagesAction))
```

#### Publishing artifacts

```scala
  val publishArtifacts = ReleaseBlock("Publishing artifacts", Seq(releaseTask(publish)))
```

#### Committing and tagging

- commit markdown documentation
- finally set and commit release version
- make a corresponding git tag


```scala
  val commitAndTag = ReleaseBlock("Committing and tagging", Seq(
    { st: State =>
      commitFiles("Autogenerated markdown documentation",
                  (Project.extract(st) get docsOutputDirs): _*)(st)
    },
    setReleaseVersion.action,
    commitReleaseVersion,
    tagRelease.action
  ), transit = true)
```

#### Publishing release on github

- push tags
- publish a Github release (notes and assets)


```scala
  val githubRelease = ReleaseBlock("Publishing release on github", Seq(
    { st: State =>
      val vcs = Project.extract(st).get(releaseVcs).
        getOrElse(sys.error("No version control system is set!"))
      vcs.cmd("push", "--tags", vcs.trackingRemote) ! st.log
      st
    },
    releaseTask(releaseOnGithub)
  ))
```

#### Setting and committing next version

```scala
  val nextVersion = ReleaseBlock("Setting and committing next version", Seq(
    setNextVersion.action,
    commitNextReleaseVersion
  ))
```

#### Pushing commits to github

```scala
  val githubPush = ReleaseBlock("Pushing commits to github", Seq(
    { st: State =>
      val vcs = Project.extract(st).get(releaseVcs).
        getOrElse(sys.error("No version control system is set!"))
      vcs.cmd("push", vcs.trackingRemote) ! st.log // pushing default branch
      vcs.cmd("push", vcs.trackingRemote, vcs.currentBranch) ! st.log // and then the current one
      st
    }
  ))
}


case object ReleaseSteps {
  import ReleaseSettings.autoImport._

  // will return None if things go wrong
  def execCommandWithState(vcs: Vcs, cmd: Seq[String], st: State): Option[State] = {

    val exitCode = vcs.cmd(cmd: _*) ! st.log

    if (exitCode == 0) Some(st) else None
  }

  // what's the point of this check??
  def isOk(vcs: Vcs): Boolean = (vcs.status !! ).trim.nonEmpty
```

### Additional release steps
This converts a task key to an action (which is implicitly converted the to a release step)

```scala
  def releaseTask[T](key: TaskKey[T]) = { st: State =>
    val extracted = Project.extract(st)
    val ref = extracted.get(thisProjectRef)
    try {
      extracted.runAggregated(key in ref, st)
    } catch {
      case e: java.lang.Error => sys.error(e.toString)
    }
  }
```

A generic action for commiting given sequence of files with the given commit message

```scala
  // NOTE: With any VCS business we always assume Git and don't care much about other VCS systems
  def commitFiles(msg: String, files: File*) = { st: State =>

    val extracted = Project.extract(st)
    val vcs = extracted.get(releaseVcs).getOrElse(sys.error("No version control system is set!"))

    def vcsExec(cmd: Seq[String]): Option[State] = execCommandWithState(vcs, cmd, st)

    val base = vcs.baseDir
```

Making paths relative to the base dir

```scala
    val paths = files map { f => IO.relativize(base, f).
      getOrElse(s"Version file [${f}] is outside of this VCS repository with base directory [${base}]!")
    }
```

adding files

```scala
    vcsExec( Seq("add", "--all") ++ paths ) flatMap {

      _ => if ( isOk(vcs) ) vcsExec( Seq("commit", "-m", msg) ++ paths) else None

    } getOrElse st
  }
```

We will need to set the version temporarily during the release (and commit it later in a separate step)

```scala
  lazy val tempSetVersion = { st: State =>
    val v = st.get(versions).getOrElse(sys.error("No versions are set! Was this release part executed before inquireVersions?"))._1
    st.log.info("Setting version temporarily to '" + v + "'")
    ReleaseStateTransformations.reapply(Seq(
      version in ThisBuild := v
    ), st)
  }
```

Almost the same as the standard release step, but it doesn't use our modified commitMessage task

```scala
  lazy val commitNextReleaseVersion = { st: State =>
    val extracted = Project.extract(st)
    val v = st.get(versions).
      getOrElse(sys.error("No versions are set! Was this release part executed before inquireVersions?"))._2
    commitFiles("Setting version to '" +v+ "'", extracted get releaseVersionFile)(st)
  }
```

Checks that you have written release notes in `notes/<version>.markdown` files and shows them

```scala
  lazy val checkReleaseNotes = { st: State =>
    val extracted = Project.extract(st)
    val v = extracted get (version in ThisBuild)
    val note: File = (extracted get baseDirectory) / "notes" / (v+".markdown")
    if (!note.exists || IO.read(note).isEmpty)
      sys.error(s"Aborting release. File [notes/${v}.markdown] doesn't exist or is empty. You forgot to write release notes.")
    else {
      st.log.info(s"\nTaking release notes from the [notes/${v}.markdown] file:\n \n${IO.read(note)}\n ")
      SimpleReader.readLine("Do you want to proceed with these release notes (y/n)? [y] ") match {
        case Some("n" | "N") => sys.error("Aborting release. Go write better release notes.")
        case _ => // go on
      }
      st
    }
  }
```

I took it from sb-release plugin, to make it strict: no releases with snapshot deps

```scala
  lazy val checkNoSnapshotDeps = { st: State =>
    val extracted = Project.extract(st)
    val ref = extracted.get(thisProjectRef)
    val (newSt, snapshotDeps) = extracted.runTask(releaseSnapshotDependencies in ref, st)
    if (snapshotDeps.nonEmpty) {
      st.log.error("Snapshot dependencies detected:\n" + snapshotDeps.mkString("\n"))
      sys.error("Aborting release due to snapshot dependencies.")
    }
    newSt
  }
```

Almost the same as the task `dependencyUpdates`, but it outputs result as a warning
and asks for a confirmation if needed

```scala
  lazy val checkDependecyUpdates = { st: State =>
    import com.timushev.sbt.updates.Reporter._
    val extracted = Project.extract(st)
    val ref = extracted.get(thisProjectRef)
    st.log.info("Checking project dependency updates...")

    val (newSt, data) = extracted.runTask(UpdatesKeys.dependencyUpdatesData in ref, st)

    if (data.nonEmpty) {
      val report = dependencyUpdatesReport(extracted.get(projectID), data)
      newSt.log.warn(report)
      SimpleReader.readLine("Are you sure you want to continue with outdated dependencies (y/n)? [y] ") match {
        case Some("n" | "N") => sys.error("Aborting release due to outdated project dependencies")
        case _ => // go on
      }
    } else newSt.log.info("All dependencies seem to be up to date")

    newSt
  }
```

Almost the same as the task `dependencyUpdates`, but it outputs result as a warning
and asks for a confirmation if needed

```scala
  lazy val checkTagList = { st: State =>
    val extracted = Project.extract(st)
    val ref = extracted.get(thisProjectRef)
    val (newSt, list) = extracted.runTask(TagListKeys.tagList in ref, st)
    if (list.flatMap{ _._2 }.nonEmpty) {
      SimpleReader.readLine("Are you sure you want to continue without fixing this (y/n)? [y] ") match {
        case Some("n" | "N") => sys.error("Aborting release due to some fixme-notes in the code")
        case _ => // go on
      }
    }
    newSt
  }
```

Announcing release blocks

```scala
  def shout(what: String, transit: Boolean = false) = { st: State =>
    val extracted = Project.extract(st)
    st.log.info("\n"+what+"\n")
    if (extracted.get(releaseStepByStep) && !transit) {
      SimpleReader.readLine("Do you want to continue (y/n)? [y] ") match {
        case Some("n" | "N") => sys.error("Aborting release")
        case _ => // go on
      }
    }
    st
  }
```

A release block is a sequence of release steps. We want this to be able
- to take their checks and run them first
- to operate on semantic groups of steps


```scala
  case class ReleaseBlock(name: String, steps: Seq[ReleaseStep], transit: Boolean = false)

  implicit def blockToCommand(b: ReleaseBlock) = Command.command(b.name){
    (b.steps map { s => s.check andThen s.action }).
      foldLeft(identity: State => State)(_ andThen _)
  }
```

This function takes a seuqence of release blocks and constructs a normal release process:
- it aggregates checks from all steps and puts them as a first release block
- then it runs `action` of every release step, naming release blocks and asking confirmation if needed


```scala
  def constructReleaseProcess(checks: ReleaseBlock, blocks: Seq[ReleaseBlock]): Seq[ReleaseStep] = {
    val allChecks = for(
        block <- blocks;
        step <- block.steps
      ) yield ReleaseStep(step.check)

    val initBlock = ReleaseBlock(checks.name, checks.steps ++ allChecks, transit = true)
    val allBlocks = initBlock +: blocks
    val total = allBlocks.length

    for(
      (block, n) <- allBlocks.zipWithIndex: Seq[(ReleaseBlock, Int)];
      heading = s"[${n+1}/${total}] ${block.name}";
      announce = ReleaseStep(shout("\n"+ heading +"\n"+ heading.replaceAll(".", "-") +"\n  ", block.transit));
      step <- announce +: block.steps
    ) yield step
  }

  val generateTestTags = Def.task {
    val file = (sourceManaged in Test).value / "tags.scala"

    IO.write(file, s"""
      |package ${releaseOnlyTestTagPackage}
      |
      |case object ${releaseOnlyTestTag.value}
      |  extends org.scalatest.Tag(${releaseOnlyTestTagPackage}.${releaseOnlyTestTag.value})
      |""".stripMargin
    )

    Seq(file)
  }

}

```




[main/scala/AssemblySettings.scala]: AssemblySettings.scala.md
[main/scala/DocumentationSettings.scala]: DocumentationSettings.scala.md
[main/scala/JavaOnlySettings.scala]: JavaOnlySettings.scala.md
[main/scala/MetadataSettings.scala]: MetadataSettings.scala.md
[main/scala/ReleaseSettings.scala]: ReleaseSettings.scala.md
[main/scala/ResolverSettings.scala]: ResolverSettings.scala.md
[main/scala/ScalaSettings.scala]: ScalaSettings.scala.md
[main/scala/TagListSettings.scala]: TagListSettings.scala.md
[main/scala/WartRemoverSettings.scala]: WartRemoverSettings.scala.md