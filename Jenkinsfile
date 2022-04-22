node {
  stage('========== Clone repository ==========') {
    checkout scm
  }
  stage('========== Build image ==========') {
    app = docker.build("haijunpark/onpremise") 
  }
  stage('========== Push image ==========') {
    docker.withRegistry('https://registry.hub.docker.com', 'github') { 
      app.push("${env.BUILD_NUMBER}")
      app.push("latest") 

            app.push("latest") 
            app.push("latest") 
      
                  app.push("latest") 
            app.push("latest") 
                  app.push("latest") 
            app.push("latest") 
                  app.push("latest") 
            app.push("latest") 
                  app.push("latest") 
            app.push("latest") 
      
    }
  }
}
