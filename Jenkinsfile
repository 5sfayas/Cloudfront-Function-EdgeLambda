//CI pipeline script
node {
     
    try {

        // Checkout all repos required for the build/deployment/tests
        stage( 'Checkout Repositories' ) {
            checkout poll: false, scm: [$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '<crednetial>', url: 'git@github.com:<yourgit>/app.git']]]
        }
        
        stage( 'Build App' ) {
            sh '''
                cd $WORKSPACE/App
                npm ci
                npm run build:dev1 
            '''
        }

        stage ('Deploy Cloudfront function'){
            sh '''
                cd $WORKSPACE/codedeploy
                echo "Running cloudfront deplyoment and .."
                aws cloudformation deploy \
                    --stack-name <cloudformation-stack-name> \
                    --template-file cloudfront.yml \
                    --s3-bucket <cloud-formation-template> \
                    --s3-prefix Stage \
                    --parameter-overrides ParameterKey=EnvType,ParameterValue=dev1 &> status.log || true

                # check if build has executed
                if [ $( grep "No changes to deploy" status.log | wc -l ) -gt 0 ]; then
                    echo "No Changes to deploy. Skiping redirect lambda deployment"
                elif [ $( grep "Successfully created" status.log | wc -l ) -gt 0 ];then
                    echo "Stack has successfully updated"
                else
                    echo "Stack Deployment failed, please check cloudformation log"
                    cat status.log
                fi             
            '''
        }

        stage( 'Deploy the build' ) {
            sh '''
                BUCKETNAME="<bucketName>"
                aws s3 sync --delete $WORKSPACE/App/dist/ s3://<bucketName>/

                # Get CloudFront ID For Invalidation
                CLOUDFRONTID=$(aws cloudfront list-distributions --query "DistributionList.Items[*].{id:Id,alias:Aliases.Items[0]}[?alias=='<your cloudfront alias>'].id" --output text)
                
                # Invalidate CDN                    
                aws cloudfront create-invalidation \
                    --distribution-id $CLOUDFRONTID \
                    --paths "/*"
            '''
        }

    } catch ( e ) {
        throw e
    }
}
