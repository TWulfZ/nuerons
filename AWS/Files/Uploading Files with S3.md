### ¿Que es un S3?
Un servicio de almacenamiento de objetos (archivos) en la nube. Está diseñado para ser masivamente escalable, duradero y de bajo costo.
### 1. Crear una Interfaz para subir archivos
Tener un lugar donde subir los archivos. (Ej: una API con Express utilizando Multer para NodeJS)
https://docs.nestjs.com/techniques/file-upload & [[Uploading Files NestJS]]

```ts
// Comment with color
@Post('file')
@UseInterceptors(FileInterceptor('file'))
uploadFile() file: Express.Multer.File) {
  return file;
}
```

### 2. Create a Bucket
![[create bucket s3.png]]

Al crear un bucket por defecto viene configurado en **privado**.
![[Pasted image 20251026144329.png]]

Y listo tenemos nuestro Bucket, pero **solo podemos acceder desde la cuenta que lo creamos**.
![[Pasted image 20251026145002.png]] 

### 3. Configurar Permisos con IAM 
Para conectar la app al bucket, creamos una identidad (un usuario o rol) y le asignamos permisos específicos.

* **IAM (Identity and Access Management):** Es el servicio que gestiona las identidades (usuarios, roles) y los permisos en AWS.
* **IAM Policy:** Es el documento (JSON) que define las reglas: qué acciones (Ej: `s3:GetObject`) puede realizar una identidad sobre un recurso (Ej: un bucket específico).

![[Pasted image 20251027082406.png]]
![[Pasted image 20251027082544.png]]

Creamos una nueva política ![[Pasted image 20251027082626.png]]
Para nuestro caso seleccionaremos para los servicios de S3
![[Pasted image 20251027082946.png]]

Damos los permisos de Lectura (`Read:GetObject`) ![[Pasted image 20251027083432.png]]
Y los permisos de escritura
![[Pasted image 20251027083550.png]]

Estos serian los permisos básicos para que funcione una aplicación web.

Ahora creamos el "scope" (alcance) que tiene el usuario, para especificarle que tenga acceso solo a un bucket.
![[Pasted image 20251027083938.png]]
Y le damos acceso solo al bucket que creamos (`planificai-s3`) con acceso a cualquiera (any) de los objetos de ese bucket (`/*`).
![[Pasted image 20251027084344.png]]
- **Nota:** Podemos darle, permisos según el directorio para hacer permisos mas granulares como `arn:aws:s3:::planificai-s3/private/*` o `arn:aws:s3:::planificai-s3/public/*`

Le damos agregar ARNs y continuamos con la creación de la política
![[Pasted image 20251027085445.png]]

Creamos la política y con un nombre y *la descripción es opcional.* ![[Pasted image 20251027085517.png]]
Y listo creamos la política de acceso
![[Pasted image 20251027085832.png]]

### 4. Crear Usuario IAM
Ahora creamos el usuario que va a utilizar nuestra API
![[Pasted image 20251027090017.png]]
![[Pasted image 20251027090342.png]]Y agregamos los permisos por política y seleccionamos la que creamos
![[Pasted image 20251027090510.png]]
No vamos a crear ningún tag así que le damos crear
![[Pasted image 20251027090705.png]]

Y listo tenemos nuestro nuevo usuario. ![[Pasted image 20251027090748.png]]
Ahora generaremos las claves de acceso del usuario
![[Pasted image 20251027092645.png]]
Y generamos para un entorno local, en este caso nuestra API
![[Pasted image 20251027092729.png]]Si queremos ponerle la descripción del tag **es opcional**.
![[Pasted image 20251027092756.png]]

Generamos la clave de acceso y pegamos las credenciales en las variables de entorno de la API, y configuramos para crear un cliente mediante un provider
![[resources/excalidraw diagrams/Get Credentials IAM AWS for S3]]
Creamos el Uploader provider, instalamos primeramente
```powershell
pnpm install @aws-sdk/client-s3
```

```ts
import { S3Client } from '@aws-sdk/client-s3';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class S3Provider {
  constructor(private configService: ConfigService) {}

  readonly s3Client = new S3Client({
    region: this.configService.getOrThrow<string>('aws.region'),
    credentials: {
      accessKeyId: this.configService.getOrThrow<string>('aws.access.key'),
      secretAccessKey: this.configService.getOrThrow<string>('aws.access.secretKey'),
    },
  });
}
```
### 5. Configurar un Servicio de Uploader
Instalamos la librería para manejar el almacenamiento de forma segura. 
```powershell
pnpm install @aws-sdk/lib-storage
```
Creamos un servicio para consumir el provider del cliente de S3, y creamos 
```ts
@Injectable()
export class UploadService {
  constructor(
    private readonly configService: ConfigService,
    private readonly s3Provider: S3Provider,
  ) {}

  private readonly bucket = this.configService.getOrThrow<string>('aws.bucketName')
  private readonly s3Client = this.s3Provider.s3Client;

  async getFiles(): Promise<File[]> {
    return this.fileRepository.find();
  }

  async uploadFile(file: Express.Multer.File): Promise<UploadResponse> {
    const key = `uploads/${Date.now()}-${file.originalname}`;

    const upload = new Upload({
      client: this.s3Client,
      params: {
        Bucket: this.bucket,
        Key: key,
        Body: file.buffer,
        ContentType: file.mimetype,
      },
    });

    try {
      const response = await upload.done();
      if (!response.Location) throw new Error('Failed to upload file');

      return {
        url: response.Location,
        publicId: key,
        size: file.size,
      };
    } catch (error) {
      throw new Error(error);
    }
  }
```
##### 📝 Notas: 
- Utiliza la fecha o algún generador random para que el nombre de los archivos sea único. 
- Es buena practica comprimir multimedia antes de enviarlas al Bucket. (Ej: `npm sharper`)
- Utilizar mejor el `Upload` del `@aws-sdk/lib-storage` que el `PutObjectCommand`  de `@aws-sdk/client-s3```
	Abstrae la lógica para evitar tener que manejar en casos de concurrencia, archivos pesados y para recomendado para Multer.

| **Característica**      | **PutObjectCommand (Doc)**  | **Upload (Tu Código)**                    |
| ----------------------- | --------------------------- | ----------------------------------------- |
| **Paquete**             | `@aws-sdk/client-s3`        | `@aws-sdk/lib-storage`                    |
| **Nivel**               | Bajo Nivel (Comando Básico) | Alto Nivel (Utilidad Gestionada)          |
| **Archivos Grandes**    | ⛔ **No** (Falla si > 5GB)   | ✅ **Sí** (Maneja Multipart Upload autom.) |
| **Concurrencia**        | No                          | ✅ **Sí** (Sube partes en paralelo)        |
| **Mejor para `Multer`** | No                          | ✅ **Sí (Recomendado)**                    |

### 6. Pre-firmar URLs temporales
Ya que el Bucket es privado debemos generar URLs publicas para que puedan ver el recurso, así que firmaremos URL temporalmente para que puedan acceder al recurso.

Para firmarlas vamos a necesitar 2 cosas:
- Bucket
- Key (Ubicación del archivo)

Creamos un método para firmar las URL
```ts
  async signUrl(key: string): Promise<string> {
    const command = new GetObjectCommand({ Bucket: this.bucket, Key: key });
    const url = getSignedUrl(this.s3Client, command, { expiresIn: 3600 });

    return url;
  }
```

Y lo podremos utilizar por ejemplo para obtener un archivo con su URL de preview.
```ts
@Injectable()
export class UploadService {
  constructor(
    @InjectRepository(File) private readonly fileRepository: Repository<File>,
    private readonly configService: ConfigService,
    private readonly s3Provider: S3Provider,
  ) {}

  private readonly bucket = this.configService.getOrThrow<string>('aws.bucketName')
  private readonly s3Client = this.s3Provider.s3Client;

  async getFile(id: number): Promise<File> {
    const file = await this.fileRepository.findOne({ where: { id: id } });
    if (!file) throw new NotFoundException('File not found');

    const signedUrl = await this.signUrl(file.key);
    return {
      ...file,
      fileUrl: signedUrl,
    };
  }
```