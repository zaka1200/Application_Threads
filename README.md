# Application_Threads

# Introduction

Ce rapport présente les résultats obtenus lors de la réalisation de l'exercice consistant à créer une application utilisant des threads pour résoudre un problème de devinette. L'objectif était de mettre en place un serveur qui génère un nombre aléatoire, puis d'interagir avec un client pour deviner ce nombre en utilisant des threads pour gérer les connexions et les échanges de messages.

# Définitions

- **Thread** : Un thread est un flux d'exécution léger qui permet d'effectuer des tâches en parallèle. Dans notre application, chaque client est géré par un thread séparé, ce qui permet au serveur de gérer plusieurs connexions simultanément.
- **Serveur** : Dans ce contexte, le serveur fait référence à l'application qui génère un nombre aléatoire et qui communique avec le client pour donner des indications sur la devinette.
- **Client** : Le client est l'application qui interagit avec le serveur en entrant des valeurs pour essayer de deviner le nombre généré.


# Serveur

Le code du serveur utilise des sockets pour gérer la communication avec les clients.

```java
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.Random;

public class Server {

    private static final int MAX_ATTEMPTS = 10;
    private int randomNumber;
    public Server() {
        // Génère un nombre aléatoire entre 0 et 10
        randomNumber = new Random().nextInt(11);
    }

```

La classe Server est responsable de la génération du nombre aléatoire et de la gestion des connexions avec les clients. Dans le constructeur de la classe, nous initialisons la variable randomNumber avec un nombre aléatoire entre 0 et 10 en utilisant la classe Random de Java.

```java
    public void start() {
        try {
            // Crée une socket serveur qui écoute sur le port 8080
            ServerSocket serverSocket = new ServerSocket(8080);
            System.out.println("Le serveur est en attente de connexion...");

            while (true) {
                // Attend une connexion d'un client
                Socket clientSocket = serverSocket.accept();
                System.out.println("Connexion établie avec le client.");

                // Crée un gestionnaire de client dans un nouveau thread
                ClientHandler clientHandler = new ClientHandler(clientSocket);
                Thread thread = new Thread(clientHandler);
                thread.start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

```

La méthode start est responsable de l'écoute des connexions des clients. Nous créons une ServerSocket qui écoute sur le port 8080. Ensuite, nous entrons dans une boucle infinie où nous attendons la connexion d'un client à l'aide de serverSocket.accept(). Une fois la connexion établie, nous créons un nouvel objet ClientHandler pour gérer le client dans un nouveau thread. Cela permet au serveur de gérer plusieurs clients simultanément.

```java
    // Retourne le nombre maximum d'essais
    public int getMaxAttempts() {
        return MAX_ATTEMPTS;
    }

```

La méthode getMaxAttempts est utilisée pour obtenir le nombre maximum d'essais depuis d'autres parties du code, notamment du côté du client.

```java
    private class ClientHandler implements Runnable {
        private Socket clientSocket;
        private int attempts;

        public ClientHandler(Socket clientSocket) {
            this.clientSocket = clientSocket;
            this.attempts = 0;
        }

        @Override
        public void run() {
            try {
                // Obtient les flux d'entrée et de sortie pour communiquer avec le client
                InputStream input = clientSocket.getInputStream();
                OutputStream output = clientSocket.getOutputStream();

                boolean guessed = false;

                while (!guessed && attempts < MAX_ATTEMPTS) {
                    // Lit la valeur envoyée par le client
                    int clientNumber = input.read();
                    attempts++;

                    // Compare la valeur avec le nombre généré et envoie un message de réponse
                    if (clientNumber < randomNumber) {
                        output.write("Saisissez un nombre plus grand".getBytes());
                    } else if (clientNumber > randomNumber) {
                        output.write("Saisissez un nombre plus petit".getBytes());
                    } else {
                        output.write("Bravo, vous avez gagné !".getBytes());
                        guessed = true;
                    }
                }

                // Envoie un message de dépassement du nombre maximum d'essais si la devinette n'est pas correcte
                if (!guessed) {
                    output.write("Désolé, vous avez dépassé le nombre maximum d'essais.".getBytes());
                }

                // Ferme la socket du client
                clientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        // Crée une instance du serveur et lance l'écoute des connexions
        Server server = new Server();
        server.start();
    }
}

```

La classe ClientHandler est une classe interne qui implémente l'interface Runnable. Elle est responsable de la gestion d'un client spécifique dans un thread séparé. Dans la méthode run, nous obtenons les flux d'entrée et de sortie pour communiquer avec le client. Ensuite, nous entrons dans une boucle où nous lisons la valeur envoyée par le client, la comparons avec le nombre généré par le serveur et envoyons un message de réponse approprié. Si le client devine correctement, nous le félicitons et la boucle s'arrête. Sinon, si le nombre maximum d'essais est atteint, nous envoyons un message de dépassement. Enfin, nous fermons la socket du client.

La méthode main crée une instance du serveur et lance l'écoute des connexions.

# Client

```java 
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;
import java.util.Scanner;

public class Client {
    public static void main(String[] args) {
        try {
            // Se connecte au serveur en utilisant l'adresse "localhost" (pour une exécution locale) et le port 8080
            Socket socket = new Socket("localhost", 8080);

            // Obtient les flux d'entrée et de sortie pour communiquer avec le serveur
            InputStream input = socket.getInputStream();
            OutputStream output = socket.getOutputStream();

            // Crée un scanner pour lire les entrées utilisateur
            Scanner scanner = new Scanner(System.in);

            // Obtient le nombre maximum d'essais du serveur
            Server server = new Server();
            int maxAttempts = server.getMaxAttempts();

```

Le code du client commence par créer une socket et se connecte au serveur en utilisant l'adresse "localhost" (pour une exécution locale) et le port 8080. Ensuite, nous obtenons les flux d'entrée et de sortie pour communiquer avec le serveur. Un scanner est également créé pour lire les entrées utilisateur. Nous obtenons également le nombre maximum d'essais du serveur en utilisant la méthode getMaxAttempts de la classe Server.

```java
            boolean won = false;
            int attempts = 0;

            while (!won && attempts < maxAttempts) {
                System.out.print("Entrez une valeur : ");
                int clientNumber = scanner.nextInt();
                attempts++;

                // Envoie la valeur au serveur
                output.write(clientNumber);

                // Lit la réponse du serveur
                byte[] buffer = new byte[1024];
                int bytesRead = input.read(buffer);
                String serverResponse = new String(buffer, 0, bytesRead);
                System.out.println(serverResponse);

                // Vérifie si le client a deviné correctement ou a dépassé le nombre maximum d'essais
                if (serverResponse.equals("Bravo, vous avez gagné !")) {
                    won = true;
                } else if (serverResponse.equals("Désolé, vous avez dépassé le nombre maximum d'essais.")) {
                    break;
                }
            }

            // Affiche un message de dépassement si le client n'a pas deviné correctement
            if (!won && attempts == maxAttempts) {
                System.out.println("Désolé, vous avez dépassé le nombre maximum d'essais.");
            }

            // Ferme la socket et le scanner
            socket.close();
            scanner.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

Le client utilise une boucle pour permettre au joueur de deviner le nombre autant de fois que possible jusqu'à ce qu'il devine correctement ou atteigne le nombre maximum d'essais. À chaque itération de la boucle, le client demande à l'utilisateur d'entrer une valeur, puis envoie cette valeur au serveur en utilisant la méthode output.write(). Ensuite, il lit la réponse du serveur à l'aide de input.read(), convertit les octets en une chaîne de caractères et l'affiche à l'utilisateur. Si le client devine correctement, il est félicité et la boucle s'arrête. Si le client dépasse le nombre maximum d'essais, il affiche un message approprié. Enfin, le client ferme la socket et le scanner.


# Résultats

Pour visualiser les résultats de l'application, veuillez consulter les captures d'écran suivantes :

## step 1:

on compile puis on execute le fichier server.java

![image](https://github.com/zaka1200/Application_Threads/assets/121964432/3615e80f-8297-4265-baee-a5294fc2631f)

## step 2:

on fait la meme chose pour le client

![image](https://github.com/zaka1200/Application_Threads/assets/121964432/6c15828d-251a-4881-aa50-635e2dbb221e)

## step 3:

apres avoir dépasser le nombre maximum d'essais le serveur envoie un msg au client

![image](https://github.com/zaka1200/Application_Threads/assets/121964432/b4e51cbc-cd79-40ff-a8cb-445fdb48fed8)

## step 4 :

on va essayer de gangner une partie 

![image](https://github.com/zaka1200/Application_Threads/assets/121964432/2f0535f4-43e8-44e1-87fd-bf625155436b)

**FIRST TRY !!**

# 
Ces captures d'écran illustrent le fonctionnement de l'application, où le client tente de deviner le nombre généré par le serveur en fonction des indications reçues.

# Conclusion

En conclusion, l'application développée en utilisant des threads a permis de résoudre l'exercice avec succès. Le serveur est capable de gérer plusieurs connexions de clients simultanément, tandis que les clients peuvent interagir avec le serveur pour deviner le nombre généré. L'utilisation de threads et de sockets offre une communication en temps réel entre le serveur et les clients, ce qui est très utile dans les scénarios nécessitant une gestion simultanée de plusieurs clients.
