This prototype app demonstrates how to create powerful data analysis tools with Shiny and [`targets`](https://docs.ropensci.org/targets/). The app manages multiple pipelines across multiple clients, and it ensures that user storage and background processes persist after logout. Because of [`targets`](https://docs.ropensci.org/targets/), subsequent runs skip computationally expensive steps that are already up to date.

## Case study

Bayesian joint models of survival and longitudinal non-survival outcomes reduce bias and describe relationships among endpoints ([Gould et al. 2015](https://pubmed.ncbi.nlm.nih.gov/24634327/)). Statisticians routinely refine and explore such complicated models ([Gelman et al. 2020](https://arxiv.org/abs/2011.01808)), but the computation is so slow that routine changes are tedious to refresh. This app shows how [`targets`](https://docs.ropensci.org/targets/) can speed up iteration and Shiny can ease the burden of code development for established use cases.

## Usage

When you first open the app, create a new project to establish a data analysis pipeline. You can create, switch, and delete projects at any time. Next, select the biomarkers and number of Markov chain Monte Carlo iterations. The pipeline will run one [univariate joint model](https://mc-stan.org/rstanarm/articles/jm.html#univariate-joint-model-current-value-association-structure) on each biomarker for the number of iterations you select. Each model analyzes [`rstanarm`](https://mc-stan.org/rstanarm/) datasets [`pbcLong`](https://mc-stan.org/rstanarm/reference/rstanarm-datasets.html) and [`pbcSurv`](https://mc-stan.org/rstanarm/reference/rstanarm-datasets.html) to jointly model survival (time to event) and the biomarker (longitudinally).

Click the "Run pipeline" button to run the correct models in the correct order. A spinner will appear in the upper right to show you that the pipeline is running in the background. The pipeline will run to completion even if you switch projects, log out, or get disconnected for idleness.

While the pipeline is running, the Progress and Logs tabs continuously refresh to monitor progress. The Progress tab uses the [`tar_watch()`](https://docs.ropensci.org/targets/reference/tar_watch.html) Shiny module, available through the functions [`tar_watch_ui()`](https://docs.ropensci.org/targets/reference/tar_watch_ui.html) and [`tar_watch_server()`](https://docs.ropensci.org/targets/reference/tar_watch_server.html).

The Results tab refreshes the final plot every time the pipeline stops. The plot shows the marginal posterior distribution of the association parameter between mortality and the longitudinal biomarker.

## Administration

1. Optional: to customize the location of persistent storage, create an `.Renviron` file at the app root and set the `TARGETS_SHINY_HOME` environment variable. If you do, the app will store projects within `file.path(Sys.getenv("TARGETS_SHINY_HOME"), Sys.getenv("USER"), ".targets-shiny")`. Otherwise, storage will default to `tools::R_user_dir("targets-shiny", which = "cache")`
2. To support persistent pipelines, deploy the app to [RStudio Server](https://rstudio.com/products/rstudio-server-pro/), [RStudio Connect](https://rstudio.com/products/connect/), or other service that supports persistent server-side storage. Alternatively, if you just want to demo the app on a limited service such as [shinyapps.io](https://www.shinyapps.io), set the `TARGETS_SHINY_TRANSIENT` environment variable to `"true"` in the `.Renviron` file in the app root directory. That way, the UI alerts the users that their projects are transient, the app writes to temporary storage (overriding `TARGETS_TRANSIENT_HOME`), and background processes terminate when the app exits.
3. Require a login so the app knows the user name.
4. Run the app as the logged-in user, not the system administrator or default user.
5. If applicable, raise automatic timeout thresholds in [RStudio Connect](https://rstudio.com/products/connect/) so the background processes running pipelines remain alive long enough to finish.

## Development

Shiny apps with [`targets`](https://docs.ropensci.org/targets/) require specialized techniques such as user storage and persistent background processes.

### User storage

[`targets`](https://docs.ropensci.org/targets/) writes to storage to ensure the pipeline stays up to date after R exits. This storage must be persistent and user-specific. This particular app defaults to `tools::R_user_dir("app_name", which = "cache")` but uses `file.path(Sys.getenv("TARGETS_SHINY_HOME"), Sys.getenv("USER"))` if `TARGETS_SHINY_HOME` is defined in the `.Renviron` file at the app root directory. In addition, it is best to deploy to a service like [RStudio Server](https://rstudio.com/products/rstudio-server-pro/) or [RStudio Connect](https://rstudio.com/products/connect/) and provision enough space for the expected number of users.

### Multiple projects

Projects manage multiple versions of the pipeline. In this app, each project is a directory inside user storage with app input settings, pipeline configuration, and results. A top-level `_project` file identifies the current active project. Functions in `R/project.R` configure, load, create, and destroy projects. The `update*()` functions in Shiny and `shinyWidgets`, such as `updateSliderInput()`, are particularly handy for restoring the input settings of a saved project. That is why this app does not need a single `renderUI()` or `uiOutput()`.

### Working directory

For reasons [described here](https://github.com/ropensci/targets/discussions/297), the `_targets.R` configuration file and `_targets/` data store always live at the root directory of the pipeline (where you run [`tar_make()`](https://docs.ropensci.org/targets/reference/tar_make.html)). So in order to run a pipeline in user storage, the app needs to change directories to the pipeline root. The best time to switch directories is when the user selects the corresponding project. This technique works as long as the Shiny server function uses a callback to restore the working directory when the app exits.

```r
server <- function(input, output, session) {
  dir <- getwd()
  session$onSessionEnded(function() setwd(dir))
  # ...
}
```

### Pipeline setup

Every [`targets`](https://docs.ropensci.org/targets/) pipeline requires a `_targets.R` configuration file and R scripts with supporting functions if applicable. The [`tar_helper()`](https://docs.ropensci.org/targets/reference/tar_helper.html) function writes arbitrary R scripts to the location of your choice, and tidy evaluation with `!!` is a convenient templating mechanism that translates Shiny UI inputs into target definitions. In this app, the functions in `R/pipeline.R` demonstrate the technique.

### Persistent background processes

The pipeline needs to run in a background process that persists after the user logs out or the app itself exits. Before you launch a new process, first check if there is already an existing process running. [`tar_pid()`](https://docs.ropensci.org/targets/reference/tar_pid.html) retrieves the ID of the most recent process to run the pipeline, and [`ps::pid()`](https://ps.r-lib.org/reference/ps_pids.html) lists the IDs of all processes currently running. If no process is already running, optionally invoke [`shinybusy::show_spinner()`](https://dreamrs.github.io/shinybusy/reference/manual-spinner.html) to indicate that the pipeline is running, then start the [`targets`](https://docs.ropensci.org/targets/) pipeline in a persistent background process:

```r
processx_handle <- tar_make(
  callr_function = r_bg,
  callr_arguments = list(
    cleanup = FALSE,
    supervise = FALSE,
    stdout = "/PATH/TO/USER/PROJECT/stdout.txt",
    stderr = "/PATH/TO/USER/PROJECT/stderr.txt"
  )
)
```

`cleanup = FALSE` keeps the process alive after the [`processx`](https://processx.r-lib.org) handle is garbage collected, and `supervise = FALSE` keeps process alive after the app itself exits. As long as the server keeps running, the pipeline will keep running. To help manage resources, the UI should have an action button to cancel the current process, and the server should automatically cancel it when the user deletes the project.

### Transient mode

For demonstration purposes, you may wish to deploy your app to a more limited service like [shinyapps.io](https://www.shinyapps.io). For these situations, consider implementing a transient mode to alert users and clean up resources. If this particular app is deployed with the `TARGETS_SHINY_TRANSIENT` environment variable equal to `"true"`, then:

1. `tar_make()` runs with the `supervise = TRUE` `callr` argument so that all pipelines terminate when the R session exits.
2. All user storage lives in a subdirectory of `tempdir()` so project files are automatically cleaned up.
3. When the app starts, the UI shows a `shinyalert` to warn users about the above.

### Progress

The [`tar_watch()`](https://docs.ropensci.org/targets/reference/tar_watch.html) Shiny module is available through the functions [`tar_watch_ui()`](https://docs.ropensci.org/targets/reference/tar_watch_ui.html) and [`tar_watch_server()`](https://docs.ropensci.org/targets/reference/tar_watch_server.html). This module continuously refreshes the [`tar_visnetwork()`](https://docs.ropensci.org/targets/reference/tar_visnetwork.html) graph and the [`tar_progress_branches()`](https://docs.ropensci.org/targets/reference/tar_progress_branches.html) table to communicate the current status of the pipeline. Visit [this article](https://shiny.rstudio.com/articles/modules.html) for more information on Shiny modules.

### Logs

The `stdout` and `stderr` log files provide cruder but more immediate information on the progress of the pipeline. To generate logs, set the `stdout` and `stderr` `callr` arguments as described previously. Then in the app server function, define text outputs that look something like this:

```r
process <- reactiveValues(running = process_running())
observe({
  invalidateLater(millis = 100)
  process$running <- process_running()
})
output$stdout <- renderText({
  req(input$project) # name of the active project
  if (process$running) invalidateLater(100)
  readLines("/PATH/TO/USER/PROJECT/stdout.txt")
})
output$stderr <- renderText({
  req(input$project)
  if (process$running) invalidateLater(100)
  readLines("/PATH/TO/USER/PROJECT/stderr.txt")
})
```

`process_running()` is a custom function that checks `targets::tar_pid()` and `ps::ps_pids()` to figure out if the pipeline is running. `process$running` is a reactive value that invalidates every time the pipeline switches from running to stopped (or vice versa) within 100 milliseconds. That way, the app only refreshes the logs if the pipeline is actually running or the user switches projects.

Lastly, define text outputs in the UI that display proper line breaks and enable scrolling:

```r
fluidRow(
  textOutput("stdout"),
  textOutput("stderr"),
  tags$head(tags$style("#stdout {white-space: pre-wrap; overflow-y:scroll; max-height: 600px;}")),
  tags$head(tags$style("#stderr {white-space: pre-wrap; overflow-y:scroll; max-height: 600px;}"))
)
```

You might also create an input switch that lets the user print only the last few lines of each log. That way, viewers can watch the logs unfold in real time as the pipeline runs.

### Results

[`targets`](https://docs.ropensci.org/targets/) stores the output of the pipeline in a `_targets/` folder at the project root. Use [`tar_read()`](https://docs.ropensci.org/targets/reference/tar_read.html) to return a result. We we want to avoid the performance costs of repeatedly reading from storage, so we use a reactive value to only do so when the pipeline starts or stops. Below, `process_running()` is a custom function that checks `targets::tar_pid()` and `ps::ps_pids()` to figure out if the pipeline is running.

```r
process <- reactiveValues(running = process_running())
observe({
  invalidateLater(millis = 100)
  process$running <- process_running()
})
output$plot <- renderPlot({
  req(input$project) # Refresh results when the user switches projects.
  process$running
  tar_read(final_plot)
})
```

## Thanks

For years, [Eric Nantz](https://shinydevseries.com/authors/admin/) has advanced the space of enterprise Shiny in the life sciences. The motivation for this app comes from his work, and it borrows many of his techniques.

## Code of Conduct
  
Please note that the `targets-shiny` project is released with a [Contributor Code of Conduct](https://contributor-covenant.org/version/2/0/CODE_OF_CONDUCT.html). By contributing to this project, you agree to abide by its terms.

## References

1. Brilleman S. "Estimating Joint Models for Longitudinal and Time-to-Event Data with rstanarm." `rstanarm`, Stan Development Team, 2020. <https://cran.r-project.org/web/packages/rstanarm/vignettes/jm.html>
2. Gelman A, Vehtari A, Simpson D, Margossian CC, Carpenter B, Yao Y, Kennedy L, Gabry J, Burkner PC, Modrak M. "Bayesian Workflow."  	*arXiv* 2020, arXiv:2011.01808, <https://arxiv.org/abs/2011.01808>.
3. Gould AL, Boye ME, Crowther MJ, Ibrahim JG, Quartey G, Micallef S, et al. "Joint modeling of survival and longitudinal non-survival data: current methods and issues. Report of the DIA Bayesian joint modeling working group." *Stat Med.* 2015; 34(14): 2181-95.
