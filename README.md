// ...

stage("config_Deploy"){
    when {
        expression { params.stages.contains("config_Deploy") }
    }
    steps {
        // Your steps for config_Deploy
    }
}

stage("Another Stage") {
    when {
        expression { 
            !params.stages.equals("config_Deploy") // Execute this stage only if config_Deploy is not the only stage selected
        }
    }
    steps {
        // Your steps for another stage
    }
}

// ...
