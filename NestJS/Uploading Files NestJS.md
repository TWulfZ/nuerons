
## Setup
Comenzamos instalando los tipos de Multer
```shell
$ npm i -D @types/multer
```

## Subir un archivo básico

```typescript
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
```
#####  Nota: Es importante, el nombre del archivo para el consumo en el Cliente. Ej.
```ts
const myFile = document.getElementById('input-file').files[0];
const formData = new FormData() as Input;

// La 'key' DEBE coincidir con la del FileInterceptor
formData.append('file', myFile); 

fetch('/upload', {
  method: 'POST',
  body: formData,
});
```
## Validar archivos
```typescript
@UploadedFile(
  new ParseFilePipeBuilder()
    .addFileTypeValidator({
      fileType: 'jpeg',
    })
    .addMaxSizeValidator({
      maxSize: 1000
    })
    .build({
      errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY
    }),
)
file: Express.Multer.File,
```

## FileFieldsInterceptor
```ts
@Post('profile')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },    
  { name: 'coverImage', maxCount: 1 } 
]))
uploadProfileFiles(
  @UploadedFiles() files: { avatar?: Express.Multer.File[], coverImage?: Express.Multer.File[] }
) {
  console.log(files.avatar[0]);    
  console.log(files.coverImage[0]); 
}
```

**En el cliente:**
```ts
const avatarFile = document.getElementById('avatar-input').files[0];
const coverFile = document.getElementById('cover-input').files[0];

const formData = new FormData();

formData.append('avatar', avatarFile);
formData.append('coverImage', coverFile);

fetch('/profile', {
  method: 'POST',
  body: formData,
});
```