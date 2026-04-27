# LAB10_security
LAB 10 : Guide d'installation de Frida

Étudiant : iliass | Machine : MyKaliLinux
Environnement : Kali Linux + Émulateur Android (x86) via ADB over TCP


1. Installation et preuve
1.1 Versions Frida
bash$ frida --version
17.9.1

$ frida-ps --version
17.9.1
Capture :

<img width="865" height="222" alt="image" src="https://github.com/user-attachments/assets/7dae71fb-6319-4d80-bb83-06a3472e1699" />


Frida 17.9.1 est correctement installé sur la machine Kali. Les deux outils frida et frida-ps sont disponibles et à jour.


1.2 Connexion ADB à l'émulateur Android
bash$ adb connect 192.168.1.214:5556
connected to 192.168.1.214:5556

$ adb devices
List of devices attached
192.168.1.214:5556    device

<img width="964" height="234" alt="image" src="https://github.com/user-attachments/assets/87cef5cf-8807-4c72-b7e3-ab04ef90f7f2" />


L'émulateur Android est accessible via ADB over TCP sur 192.168.1.214:5556. Le device est bien reconnu et en état device (prêt).


2. Déploiement Android
2.1 Extraction et push du serveur Frida
bash# Extraction du binaire compressé
$ unxz frida-server-17.9.1-android-x86.xz

# Push vers le device via ADB
$ adb -s 192.168.1.214:5556 push frida-server-17.9.1-android-x86 /data/local/tmp/frida-server
frida-server-17.9.1-android-x86: 1....3 MB/s (53658384 bytes in 1.744s)

# Attribution des permissions d'exécution
$ adb -s 192.168.1.214:5556 shell chmod 755 /data/local/tmp/frida-server
2.2 Démarrage du serveur Frida en root
bash$ adb -s 192.168.1.214:5556 shell
generic_x86_arm:/ $ su root
generic_x86_arm:/ # /data/local/tmp/frida-server &
[1] 6949
generic_x86_arm:/ #

<img width="1192" height="506" alt="image" src="https://github.com/user-attachments/assets/7654e946-e24c-4ed5-896c-9f0c616c1359" />


Le serveur Frida tourne en arrière-plan (PID 6949) avec les droits root sur l'émulateur Android x86.


3. Injection — Hook sur com.pwnsec.firestorm
3.1 Script frida_firestorm.js
Le script frida_firestorm.js a été chargé et injecté dans l'application com.pwnsec.firestorm via le tunnel Frida sur 127.0.0.1:27042.
bash$ frida -H 127.0.0.1:27042 -f com.pwnsec.firestorm -l frida_firestorm.js
3.2 Résultat de l'injection
Spawned `com.pwnsec.firestorm`. Resuming main thread!
[Remote::com.pwnsec.firestorm ]→ [*] Script chargé
[*] Hook installé ✅
[Remote::com.pwnsec.firestorm ]→
[Remote::com.pwnsec.firestorm ]→ exit

Thank you for using Frida!

<img width="1177" height="556" alt="image" src="https://github.com/user-attachments/assets/0df884c2-d48a-4944-b375-ecbb32474abd" />


L'application a été spawnée, le script chargé avec succès, et le hook installé. La session s'est terminée proprement après exit.


4. Dépannage — Simulation d'une erreur et diagnostic
4.1 Contexte : Simulation de l'erreur
Après avoir testé le hook avec succès, le serveur Frida a été arrêté intentionnellement depuis le shell de l'émulateur pour simuler une panne.
bash# Sur l'émulateur : arrêt forcé du serveur Frida
generic_x86_arm:/ # kill 6949
[1]+  Terminated    /data/local/tmp/frida-server

4.2 Symptôme observé
En relançant une commande Frida depuis Kali après l'arrêt du serveur :
bash$ frida -H 127.0.0.1:27042 -f com.pwnsec.firestorm -l frida_firestorm.js
Erreur obtenue :
Failed to connect to remote frida-server: unable to connect to 127.0.0.1:27042

4.3 Diagnostic étape par étape
Étape 1 — Vérifier que le device ADB est toujours connecté
bash$ adb devices
List of devices attached
192.168.1.214:5556    device
✅ Le device est toujours disponible — le problème vient du serveur Frida, pas d'ADB.
Étape 2 — Vérifier si frida-server tourne encore
bash$ adb -s 192.168.1.214:5556 shell "ps -A | grep frida"
# (aucune sortie)
❌ Aucun processus frida-server actif — confirmé que le serveur est arrêté.
Étape 3 — Vérifier le port de forwarding ADB
bash$ adb -s 192.168.1.214:5556 forward tcp:27042 tcp:27042
27042

Cette commande est nécessaire si le forwarding a été réinitialisé. Elle relie le port local 27042 au port du serveur Frida sur l'émulateur.


4.4 Correction
bash# Redémarrer le serveur Frida sur l'émulateur
$ adb -s 192.168.1.214:5556 shell
generic_x86_arm:/ $ su root
generic_x86_arm:/ # /data/local/tmp/frida-server &
[1] 7103
generic_x86_arm:/ #
Puis relancer l'injection :
bash$ frida -H 127.0.0.1:27042 -f com.pwnsec.firestorm -l frida_firestorm.js
Résultat :
Spawned `com.pwnsec.firestorm`. Resuming main thread!
[Remote::com.pwnsec.firestorm ]→ [*] Script chargé
[*] Hook installé ✅
✅ Le hook est à nouveau opérationnel.

4.5 Résumé du diagnostic
ÉtapeCommandeRésultatVérifier ADBadb devicesDevice toujours connectéVérifier frida-serverps -A | grep fridaProcessus absent → serveur arrêtéReforwarder le portadb forward tcp:27042 tcp:27042Port rétabliRedémarrer le serveur/data/local/tmp/frida-server &PID 7103 actifRelancer l'injectionfrida -H 127.0.0.1:27042 -f ...Hook installé ✅

Environnement
ComposantVersion / ValeurFrida (host)17.9.1frida-server (Android)17.9.1-android-x86ADB target192.168.1.214:5556Architecture émulateurx86_armApplication cibléecom.pwnsec.firestormTunnel Frida127.0.0.1:27042
