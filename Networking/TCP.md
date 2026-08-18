
All the services like ssh, databases, web servers, FTP heavily rely on TCP Protocol.

### **Transmission Control Protocol :**

Protocol which controls transmission of data.
It transmit data in a controls manner traveling from source to destination.

![[Pasted image 20260818225200.png]]

1. **TCP is a Layer 4 Protocol.** **Layer 4 is Transport Layer.**

2. Similar to UDP, TCP also uses port numbers to identify applications on the host machine. This port number information is carried via segment in TCP. Segment is TCP's data unit.

3. It is a connection oriented protocol unlike UDP(Connection less Protocol). 
   TCP needs to establish a connection between sender and receiver in order to send data over the network.
   TCP is a connection oriented protocol and it uses 3 way handshake to establish a connection. (Sync-> Sync Acknowledged->  Acknowledged)
   ![[Pasted image 20260818224929.png]]
4. TCP is a stateful protocol. It stores some data like sequence numbers, window size, acknowledgment etc, of the other party on the host machine, maintains some kind of state that's why it is stateful protocol.
   
5. TCP uses 4-way termination process (FIN/ACK) to gracefully close TCP connections. 
   
6. *Data is divided into segments before transmission.* 
   A HTTP request (stream of bytes) is break down into chunks of data to send it reliably because it can be large data.
   Instead of sending large data at once, TCP breaks this into chunks & does this chunking internally. 
   And this chunk is called TCP Segment.
   ![[Pasted image 20260818230229.png]]
7. Segments provides reliable, ordered of data between applications. When these segments travels over the network each segment is embedded in IP Packets and there is no guarantee which segment will reach the destination first (Maybe segment 5 will reach destination first in above diagram).
   TCP guarantee the order to be rearranged at destination even if the order off segment arrival is unordered.
   If a segment is dropped midway, TCP will detect it and will retransmit it unless it reach destination. 
   TCP uses acknowledgments for figuring out if packet/segment is dropped or not and retransmit it. **It covers Reliability and ordered part of TCP**.
   
8. Receiver sends acknowledgments (ACKs) to confirm received data. 
   If segment 1 reaches the destination destination will tell source that I got segment 1 (ACK).
   This all process is in built in TCP.
   
9. Lost segments are detected & transmitted. (Read point 7)
   
10. TCP has overhead of all the above things so TCP is bit slow than UDP. But it is more reliable than UDP. We cannot loose the data and it should be in order as well.


## TCP Usecases
![[Pasted image 20260818234718.png]]
1. All REST APIs uses HTTP. HTTP under the hood uses TCP. 
![[Pasted image 20260819000137.png]]