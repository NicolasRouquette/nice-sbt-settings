## Documentation settings

This module takes care of producing two kinds of documentation:

- Converting sources to markdown with [literator](https://github.com/laughedelic/literator)
- Generating API docs (javadocs/scaladocs) and pushing it to the gh-pages branch


```scala
package ohnosequences.sbt.nice

import sbt._
import Keys._

import sbtrelease._, ReleasePlugin.autoImport._

import laughedelic.literator.plugin.LiteratorPlugin.autoImport._

case object DocumentationSettings extends sbt.AutoPlugin {

  override def requires = laughedelic.literator.plugin.LiteratorPlugin
  override def trigger = allRequirements

  case object autoImport {
```

### Commands

Commands are added for convenience of invoking these actions manually from sbt repl


```scala
    lazy val cleanAndGenerateDocs = Command.command("cleanAndGenerateDocs")(cleanAndGenerateDocsAction)
    lazy val pushApiDocsToGHPages = Command.command("pushApiDocsToGHPages")(pushApiDocsToGHPagesAction)
  }
  import autoImport._
```

### Settings

```scala
  override lazy val projectSettings: Seq[Setting[_]] = Seq(
    commands ++= Seq(
      cleanAndGenerateDocs,
      pushApiDocsToGHPages
    )
  )
```

### Actions

Actions (`State => State` functions) are nice because you can make from them commands and they
can be directly used as release steps

This action _cleans_ docs out directory first and then generates docs (but doesn't commit)

```scala
  lazy val cleanAndGenerateDocsAction = { st: State =>
    val extracted = Project.extract(st)
    val ref = extracted.get(thisProjectRef)
    Defaults.doClean(extracted get docsOutputDirs, Seq())
    extracted.runAggregated(generateDocs in ref, st)
  }
```

This action tries
- clone `gh-pages` branch
- generate api docs with the standard sbt task `docs` to the `docs/api/<version>/` dir
- commit it and push the `gh-pages` branch


```scala
  lazy val pushApiDocsToGHPagesAction = { st: State =>
    val extracted = Project.extract(st)
    val ref = extracted.get(thisProjectRef)

    extracted get releaseVcs match {
      case None => sys.error("No version control system is set!")
      case Some(vcs) => {
        lazy val remote: String = vcs.cmd("config", "branch.%s.remote" format vcs.currentBranch).!!.trim
        if (remote.isEmpty) sys.error("Remote branch for ${vcs.currentBranch} is not set up properly")

        lazy val url: String = vcs.cmd("ls-remote", "--get-url", remote).!!.trim
        if (url.isEmpty) sys.error("Remote url is not set up properly")

        val ghpagesDir = IO.createTemporaryDirectory
        if (vcs.cmd("clone", "-b", "gh-pages", "--single-branch", url, ghpagesDir).! != 0) {
          st.log.error("Couldn't generate API docs, because this repo doesn't have gh-pages branch")
          st.log.error("Create the branch and rerun the [pushApiDocsToGHPages] command")
          st
        } else {
          val newSt = ReleaseStateTransformations.reapply(Seq(
              target in (Compile, doc) := ghpagesDir / "docs" / "api" / extracted.get(version).stripSuffix("-SNAPSHOT")
            ), st)
          val lastSt = Project.extract(newSt).runAggregated(doc in Compile in ref, newSt)

          val ghpages = new Git(ghpagesDir)
          ghpages.cmd("add", "--all", "docs/api") ! lastSt.log
          ghpages.commit("Updated API docs for sources commit: " + vcs.currentHash) ! lastSt.log
          ghpages.cmd("push") ! lastSt.log

          lastSt
        }
      }
    }
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