stage("config_Deploy"){
    when {
        expression { params.stages.contains("config_Deploy") }
    }
    steps {
        // Your steps for config_Deploy
    }
}
