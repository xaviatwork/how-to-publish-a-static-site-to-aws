# Construir y desplegar documentación en AWS

El objetivo de esta guía es describir cómo crear documentación como código y publicarla en AWS.

Usamos *documentación como código* porque el *código fuente* de la documentación son documentos de texto plano en formato [markdown](https://daringfireball.net/projects/markdown/) que se almacenan en un sistema de control de versiones.

Cada vez que se produzca una actualización de la documentación (un nuevo *commit* en el repositorio), se lanza una *pipeline* que construye y publica la documentación en la web.

Al tratarse de una solución para la publicación de documentación interna del departamento, uno de los condicionantes es reducir al máximo la gestión de la infraestructura necesaria para mantener y publicar la documentación.

Con esta idea, se usarán servicios gestionados *serverless* siempre que es posible:

- **AWS CodeCommit** como repositorio de código basado en Git
- **AWS CodeBuild** para convertir la documentación de markdown a HTML

## Contruyendo la documentación

Usaremos [MkDocs](https://www.mkdocs.org/) para general la documentación en formato HTML a partir de los ficheros en formato markdown.

Para construir los ficheros HTML, usamos CodeBuild.

CodeBuild usa un fichero de configuración llamado `buildspec.yml` que debe ubicarse en la raíz del repositorio.

```yaml
# pseudo-code + Untested
version: 0.1

phases:
    install:
        commands:
            - install mkdocs
            - install mkdocs-material
    build:
        commands:
            - mkdocs build
    deploy:
        commands:
            - s3 cp . s3://bucket/docs-site
```

En AWS CodeBuild, creamos un nuevo proyecto con la configuración:

```yml
Project name: 'docs'
Source provider: 'codecommit'
Repository: 'repo-name'
Webhook: 'No'
Environment image: 'Use an image managed by AWS CodeBuild'
Operating system: 'Ubuntu'
Runtime: 'Python'
Runtime version: 'aws/codebuild/python:3.6'
Artyfacts type: 'No artifacts'
Cache type: 'No cache'
Service role: 'Create a service role in your account'
VPC: No VPC
```

CodeBuild debe ejecutarse con permisos suficientes para realizar las tareas indicadas. Para ello, creamos un rol y asignamos:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Resource": [
                "arn:aws:logs:eu-west-1:xxxxxxxxxxxx:log-group:/aws/codebuild/www_zeta-two_com",
                "arn:aws:logs:eu-west-1:xxxxxxxxxxxx:log-group:/aws/codebuild/www_zeta-two_com:*"
            ],
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ]
        },
        {
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::codepipeline-eu-west-1-*"
            ],
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:GetObjectVersion"
            ]
        }
    ]
}
```

También necesitamos que CodeBuild edite el contenido del bucket S3. Para ello, usamos la política de buckets del sitio para:

```json
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "PermissionsForRoleOnTheBucket",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::xxxxxxxxxxxx:role/service-role/<codebuild-role>"
            },
            "Action": [
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::<bucketName>/*"
        },
        {
            "Sid": "PermissionsForRoleOnTheBucketListBucketContents",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::xxxxxxxxxxxx:role/service-role/<codebuild-role>"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<bucketName>"
        }
    ]
}
```

La pieza final que pone en marcha toda la cadena es un proyecot en CodePipeline:

```yaml
Pipeline name: 'docs'
Source provider: 'CodeCommit'
Repository: '<repositoryName>'
Branch: 'master'
Build provider: 'AWS CodeBuild'
'Select an existing build project'
Project name: '<projectName>'
Deployment provider: 'No deployment'
Role name: 'AWS-CodePipeline-service'
```

Si todo funciona correctamente, al hacer un nuevo *commit* y hacer *push* contra la rama `master`, una nueva versión del sitio web con la documentación debería desplegarse.

---

Referencia: [Static Jekyll site with S3, CloudFront & CodePipeline](https://zeta-two.com/software/2018/07/09/static-website-s3-codepipeline.html)
