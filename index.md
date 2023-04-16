---
title: Informe DSI P10
subtitle: API's asíncronas: ficheros, procesos, sockets
---

# Tareas previas


# Ejercicios

# Ejercicio 1 - El alergólogo

## Enunciado 


## Implementación
### Desarrollo de pruebas para la metodología TDD

Se contempla la siguiente estructura base:


## Resultado

Se ha implementado un algoritmo recursivo que devuelve la primera opción de suma válida (de mayor identificación del alérgeno a menor).

Ejemplo de ejecución:

![Ej1 Resultado Ejecución](Assets/Imgs/Ej1Result.png)
# Ejercicio 2 - Números complejos

## Enunciado

Defina un tipo de datos para números complejos y escriba funciones para operaciones básicas (suma, resta, multiplicación, división), producto escalar, conjugado y módulo.

## Implementación
## Desarrollo de pruebas para la metodología TDD

Se contempla la siguiente estructura base:
```
/**
 * Definición del tipo de datos para el número completo, consistente en una tupla
 */
type ComplexNumber = [number, number];

/**
 * Definición de los índices de la parte real e imaginaria del número complejo
 */
export enum ComplexIndex {
  Real = 0,
  Imaginary = 1,
}

/**
 * Función para sumar dos números complejos
 * @param complexNum1 Primer Número Complejo
 * @param complexNum2 Segundo Número Complejo
 * @returns Un número complejo que representa la suma de los números complejos pasados por parámetro
 */
export function add(complexNum1: ComplexNumber, complexNum2: ComplexNumber): ComplexNumber | undefined {
  let result: ComplexNumber;
  // Implementación de la suma de los números complejos
  return result;
}
…
```
Que deberá cumplimentar con éxito las siguiente pruebas y condiciones:

![Ej2 TDD](Assets/Imgs/Ej2TDD.png)

## Resultado

Ejemplo de ejecución:

![Ej2 Resultado Ejecución](Assets/Imgs/Ej2Result.png)

# Ejercicio 3 - Cliente y servidor para aplicación de registro de Funko Pops

## Enunciado

Se deberá implementar un servidor y un cliente que usen sockets utilizando el módulo net de Node.js para la aplicación de registro de Funko Pops de la práctica anterior. El cliente solicita las mismas operaciones que se implementaron, y la información se deberá almacenar en ficheros JSON en el sistema de ficheros del servidor. El usuario interactúa con el cliente a través de la línea de comandos utilizando yargs.

## Implementación - Novedades

Los ficheros Funko.ts y FunkoCollectionManager.ts propios de la práctica 9 y que incluyen la implementación de clases y operaciones del objeto Funko no han sido prácticamente modificados. El único cambio que se ha realizado es que el resultado de las operaciones (add, update, show...) ya no imprime el resultado por pantalla. En su lugar estas funciones devuelven una string con el resultado que será recogido por el servidor para enviarlo como respuesta al cliente en formato JSON.

## Clase messageEventEmitter

```
import net from 'net';
import { EventEmitter } from 'events';

export type messageEventEmitterTypeOptions = {
  emitterType: 'client' | 'server';
}

export class MessageEventEmitter extends EventEmitter {
  constructor(public connection: net.Socket, public emitterType?: messageEventEmitterTypeOptions) {
    super();

    let wholeData = '';
    connection.on('data', (dataChunk) => {
      wholeData += dataChunk;

      let messageLimit = wholeData.indexOf('\n');
      while (messageLimit !== -1) {
        const message = wholeData.substring(0, messageLimit);
        wholeData = wholeData.substring(messageLimit + 1);
        if (this.emitterType?.emitterType === 'client') {
          this.emit('response', JSON.parse(message));
        } else if (this.emitterType?.emitterType === 'server') {
          this.emit('request', JSON.parse(message));
        } else {
          this.emit('message', JSON.parse(message));
        }
        messageLimit = wholeData.indexOf('\n');
      }
    });
  }

  public write(type: string, message?: string) {
    this.connection.write(JSON.stringify({ type: type, message: message }) + '\n');
  }
}
```
La clase MessageEventEmitter extiende de la clase EventEmitter de Node.js. Esta clase se utiliza para leer y escribir mensajes JSON en un socket de red (ya sea de un cliente o un servidor). La clase MessageEventEmitter se inicializa con un objeto connection de tipo net.Socket y una cadena emitterType que puede ser 'client' o 'server', esto es importante porque según esta configuración se enviarán eventos tipo 'request' o 'response'.

En el constructor de la clase, se escucha el evento data en el socket de red. Cada vez que se recibe un fragmento de datos, se agrega al final de una cadena wholeData que contiene los datos completos recibidos hasta el momento. Luego, se busca el índice del carácter de nueva línea ('\n') en la cadena wholeData. Si se encuentra, se extrae el mensaje JSON desde el principio de la cadena hasta el índice del carácter de nueva línea y se emite un evento ('response', 'request' o 'message') dependiendo del tipo de emisor definido con emitterType.

También tiene un método write que se utiliza para enviar mensajes JSON al socket de red. Este método toma dos argumentos: una cadena type y una cadena message. El método convierte estos argumentos en un objeto JSON y lo envía al socket de red con un carácter de nueva línea al final.

## Servidor de la aplicación

```
import net from 'net';
import chalk from 'chalk';
import { MessageEventEmitter, messageEventEmitterTypeOptions} from './messageEventEmitter.js';
import { FunkoCollectionManager } from './FunkoCollectionManager.js';
import { Funko, FunkoGenre, FunkoType } from './Funko.js';

net.createServer((socket) => {
    const emitterType: messageEventEmitterTypeOptions = { emitterType: 'server'}
    const server = new MessageEventEmitter(socket, emitterType);
    console.log(chalk.green('A client has connected.'));

    server.write("validConnection");

    server.on('request', (request) => {
      try {  
        const inputMessage = JSON.parse(request.message.toString());
        const manager = new FunkoCollectionManager(inputMessage.user);
  
        if (request.type === 'add') {
          const newFunko = new Funko("", "", "", FunkoType.POP, FunkoGenre.ANIMATION, "", 0, false, "", 0).parse(inputMessage.funkoData);
          const outputMessage = manager.addFunko(newFunko);
          server.write('add', JSON.stringify({output: outputMessage}));
        } else if (request.type === 'update') {
          const newFunko = new Funko("", "", "", FunkoType.POP, FunkoGenre.ANIMATION, "", 0, false, "", 0).parse(inputMessage.funkoData);
          const outputMessage = manager.modifyFunko(inputMessage.funkoID, newFunko);
          server.write('update', JSON.stringify({output: outputMessage}));
        } else if (request.type === 'read') {
          const outputMessage = manager.showFunko(inputMessage.funkoID);
          server.write('read', JSON.stringify({output: outputMessage}));
        } else if (request.type === 'list') {
          const outputMessage = manager.listFunkos();
          server.write('list', JSON.stringify({output: outputMessage}));
        } else if (request.type === 'remove') {
          const outputMessage = manager.removeFunko(inputMessage.funkoID);
          server.write('remove', JSON.stringify({output: outputMessage}));
        } else {
          console.log(chalk.red(`Invalid request type ${request.type}`));
        }
        server.connection.end();
      } catch (error) {
        console.error(chalk.red(`Invalid JSON message received: ${request.message.toString()}`));
        server.write('error', JSON.stringify({output: chalk.red(`Invalid JSON message sended: ${request.message.toString()}`)}));
        server.connection.end();
      }
    });

    server.connection.on('close', () => {
      console.log(chalk.green('A client has disconnected.'));
    });
  }).listen(60300, () => {
    console.log(chalk.green('Waiting for clients to connect.'));
});
```

Este servidor de red estará escuchando el puerto 60300. Cuando se establece una conexión, el servidor crea una nueva instancia de MessageEventEmitter, que hereda de la clase EventEmitter, para gestionar los eventos y mensajes que se envían y reciben entre el cliente y el servidor.

El servidor responde con un mensaje "validConnection" cuando se establece la conexión. Luego, espera mensajes entrantes del cliente (eventos de tipo 'request', que son de tipo JSON y contiene información sobre las operaciones que el cliente solicitará realizar.

Cuando se recibe una 'request', se crea una instancia de FunkoCollectionManager y se realiza la operación correspondiente en la colección de figuras de Funko. Luego se envía una respuesta de vuelta al cliente con los resultados de la operación.

Si se produce un error al analizar el mensaje JSON o al procesar la operación solicitada, el servidor envía una respuesta de error al cliente y cierra la conexión. Si la operación se realiza correctamente, se cierra la conexión después de enviar la respuesta al cliente. En cualquier caso el servidor seguirá estando activo escuchando el puerto en busca de nuevas conexiones entrantes.

## Cliente de la aplicación

```
import {connect} from 'net';
import yargs from 'yargs';
import chalk from 'chalk';
import { hideBin } from 'yargs/helpers';
import { IFunkoData } from './Funko.js';
import {MessageEventEmitter, messageEventEmitterTypeOptions} from './messageEventEmitter.js';

const emitterType: messageEventEmitterTypeOptions = { emitterType: 'client' }
const client = new MessageEventEmitter(connect({port: 60300}), emitterType);



client.on('error', (error) => {
  console.error(chalk.red('Error in client connection:'), error);
});

client.on('response', (response) => {
  if (response.type === 'validConnection') {
    console.log(chalk.green(`Connection established`));
  } else if (response.type === 'add' || 'update' || 'remove' || 'read' || 'list') { 
    const output = JSON.parse(response.message.toString());
    console.log(output.output);
  } else {
    console.error(chalk.red(`Message type ${response.type} is not valid`));
  }
});

const argv = yargs(hideBin(process.argv)) 
...
```

El cliente es una aplicación que utiliza la línea de comandos mediante yargs para interactuar con el servidor y el sistema de Funkos que este almacena. A través de la línea de comandos el usuario podrá agregar, actualizar, eliminar, leer o listar elementos Funko del sistema.

El cliente es también un objeto de MessageEventEmitter, en este caso el emitterType es 'client'. Al instanciar el cliente se pasa como argumento un net.Socket resultado de conectarse al puerto 60300.

Cuando se recibe una respuesta del servidor, la aplicación verifica el tipo de mensaje recibido. Si es "validConnection", se muestra un mensaje de conexión exitosa en verde. Si el tipo de mensaje es "add", "update", "remove", "read" o "list", se analiza la respuesta, que está en formato JSON, y se muestra el resultado. Si el tipo de mensaje no es uno de los anteriores, se muestra un mensaje de error en rojo.

## Ejemplo de ejecución


# Referencias
* [Vídeo Instalación y uso de TypeDoc](https://drive.google.com/file/d/19LLLCuWg7u0TjjKz9q8ZhOXgbrKtPUme/view/)
* [Documentación de TypeDoc](https://typedoc.org/guides/doccomments/)
* [Vídeo Instalación y uso de Mocha y Chai](https://drive.google.com/file/d/1-z1oNOZP70WBDyhaaUijjHvFtqd6eAmJ/view/)
* [Documentación de Chai](https://www.chaijs.com/)
* [Arrays, tuplas y enumeradores](https://ull-esit-inf-dsi-2223.github.io/typescript-theory/typescript-arrays-tuples-enums.html/)
* [Enunciado Práctica](https://ull-esit-inf-dsi-2223.github.io/prct04-arrays-tuples-enums/)
