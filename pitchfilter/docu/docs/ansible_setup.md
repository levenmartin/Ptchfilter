
#Ansible Setup

Als erstes werden alle noch laufenden jackTrip-Prozesse auf den Zielhosts beendet, damit vorherige JackTrip-Verbindungen ausgeschlossen werden lönnen.
Ein JackTrip Server wird auf den Master-Knoten gestartet, welcher als AP-05 defiert wird.
Dann werden viele JackTrip-Clients gestartet, die sich mit dem Master-Knoten Verbinden.
Dadaurch entsteht ein vollständig verbundener Jacktrip-Mesh.



```
---
- name: Start fully connected Jacktrip Mesh
  hosts: sprawl_nodes
  gather_facts: false
  tasks:
    - name: Kill all JackTrip things
      command: killall jacktrip
      ignore_errors: true
      #Hier werden alle laufenden JackTrip-Prozesse auf den Zielhosts beendet.

    - name: "Launch JackTrip Server"
      shell: jacktrip -S -p5
      async: 2592000 # run for 1 month
      poll: 0
      when: inventory_hostname == master_node
      #startet einen JackTrip-Server auf dem Master-Knoten

    - name: "Launch lots of JackTrip clients"
      # create connection to server with the name
      shell: jacktrip -n 1 -C {{ master_node }} -K {{ inventory_hostname }} -J {{master_node}}
      async: 2592000 # run for 1 month
      poll: 0
      when: inventory_hostname != master_node
      #Dieser Abschnitt sorgt dafür, dass viele JackTrip-Clients gestartet werden und diese sich mit dem JackTrip-Server verbinden.
      

  vars:
    base_port: 4464
    master_node: AP-05

```







In diesem Abschnitt wird das System vorbereitet, indem verschiedene Playbooks importiert werden und Aufgaben ausgeführt werden.
Dadurch werden verbindungen gelegt und das System so in den richtigen Zustand versetzt.


```
---
- name: Start Pitchfilter Supercollider
  ansible.builtin.import_playbook: pitchfilter_supercollider.yml
  #Supercollider Datei wird gestartet

- name: Setup Jacktrip Mesh
  ansible.builtin.import_playbook: ../../pi_setup/playbooks/launch_jacktrip_mesh.yml
  #Jacktrip-Mesh-Netzwerk wird eingerichtet

- name: "Sleepytime"
  hosts: localhost
  gather_facts: false
  tasks:
    - name: GuNa
      ansible.builtin.wait_for:
        timeout: 4
        #Wartezeit wird auf 4 sekunden gesetzt

- name: Setup Connections
  ansible.builtin.import_playbook: pitchfilter_connect.yml
  #Playbook zum festlegen der Verbindungen wird eingerichet
```


In diesem Abschnit werden die verbindungen zwischen Jack-Audio-System und Supercollider gelegt.

Dabei wird das ausgehende Audiosignal des Masterknoten an den Supercollider Eingang in_1 jedes Knoten gesendet der nicht der Masterknoten ist.
```
---
- name: Setup Jack Connections
  hosts: sprawl_nodes
  gather_facts: false
  #Playbook wird auf der Gruppe "sprawl_nodes" ausgeführt
  vars: 
    master_node: AP-05
    #Master-Knoten wird bestimmt
  tasks:
    - name: Connect local ins/outs
      shell: |
        jack_connect system:capture_1 SuperCollider:in_1
        jack_connect system:capture_2 SuperCollider:in_1
        jack_connect {{master_node}}:receive_1 SuperCollider:in_2
        jack_connect SuperCollider:out_1 system:playback_1
      when: master_node != inventory_hostname
      #Stellt lokale Audioverbindungen zwischen System und SuperCollider her, falls der Masterknoten nicht der aktuelle Knoten ist.

    - name: Connect jacktrip clients
      shell: |
        jack_connect system:capture_1 {{ item }}:send_1
        jack_connect system:capture_2 {{ item }}:send_1
      loop: "{{ ansible_play_hosts | difference([inventory_hostname]) }}"
      when: master_node == inventory_hostname
      #Stellt Verbindungen zwischen JackTrip-Clients und dem System her, falls der Masterknoten der aktuelle Knoten ist.

```


Zusammengefasst bereitet dieser Abschnitt das System vor, kopiert erforderliche Dateien, beendet vorherige SuperCollider-Prozesse (falls vorhanden) und startet SuperCollider mit bestimmten Konfigurationen und einem langen Laufzeitintervall.

```
---
- name: Start Supercollider
  hosts: sprawl_nodes
  gather_facts: false
  tasks:
    - name: "Ensure 'pieces' dir exists"
      ansible.builtin.file:
        path: /home/member/pieces/
        state: directory
        owner: member
        group: member
        mode: "u=rwx,g=rx,o=rx"
      # Stellt sicher, dass das Verzeichnis 'pieces' vorhanden ist und die richtigen Berechtigungen hat.

    - name: "Copy Files onto the server"
      copy:
        src: SC
        dest: /home/member/pieces/pitchfilter/
        owner: member
        group: member
        mode: "0644"
      # Kopiert Dateien aus dem 'SC'-Verzeichnis auf den Server in das Verzeichnis '/home/member/pieces/pitchfilter/' mit den erforderlichen Berechtigungen.

    - name: "Kill sclang"
      shell: killall sclang
      ignore_errors: true
      # Beendet alle laufenden 'sclang'-Prozesse, falls vorhanden.

    - name: "Kill scsynth"
      shell: killall scsynth
      ignore_errors: true
      # Beendet alle laufenden 'scsynth'-Prozesse, falls vorhanden.

    - name: "Launch SC!"
      async: 2592000 # run for 1 month
      poll: 0
      shell: DISPLAY=:0 sclang main.scd >> /tmp/pitchfilter.log
      args:
        chdir: /home/member/pieces/pitchfilter/SC
      # Startet SuperCollider und führt das Skript 'main.scd' aus. Der Prozess läuft für einen Monat und die Ausgabe wird in '/tmp/pitchfilter.log' protokolliert.

```