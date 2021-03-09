# PTC_Projeto1
[Projeto1: protocolo de enlace](https://moodle.ifsc.edu.br/course/view.php?id=7422#section-3)


O protocolo de enlace a ser desenvolvido implementa um serviço com estas características:
* protocolo de enlace ponto-a-ponto para um canal sem-fio, e que usa uma camada física do tipo UART
* encapsulamento de mensagens com até 1024 bytes
* recepção de mensagens livres de erros
* garantia de entrega com mecanismo ARQ
* controle de acesso ao meio
* conectado (estabelecimento de sessão)

O desenvolvimento do protocolo da sua equipe será realizado aqui no Github. Seguem orientações:
* Use [Issues](issues) para comunicação com o professor, e entre os membros da equipe. Por ali muitas dúvidas poderão ser resolvidas em relação ao seu projeto !
* Organize seu projeto em quadros usando [Projects](projects). Define um quadro para tarefas a iniciar, outra para tarefas em andamento, e uma terceira para tarefas concluídas. Se achar melhor, organize suas tarefas de outras maneiras ali. Aplique o que aprendeu em Projeto Integrador II !
* Fique atento a prazos ! Veja sempre no Moodle as tarefas do projeto, e procure acompanhar o ritmo dos trabalhos. Se tiver problemas, contate o professor.
<br/>


# Máquina de estados do enquadramento
<br/>
<p align="center"> <a href="https://github.com/Joaovitorrod/PTC_Projeto1/blob/master/Resources/state_machines/Enquadramento/exercicio_maq_estados_PTC.pdf">
  <img src="https://github.com/Joaovitorrod/PTC_Projeto1/blob/master/Resources/state_machines/Enquadramento/exercicio_maq_estados_PTC.png" width="700" title="Máquina de estados Enquadramento">
</p><br/><br/>











import sys
from Quadro import *
from enum import Enum
import attr

from Layer import *

IPV4 = 0x800
IPV6 = 0x86dd

ACK = 0
NACK = 1


@attr.s
class Evento:
    'Representa um evento, contendo seu tipo e os dados a ele associados'
    tipo = attr.ib(default=TipoEvento.Timeout)


class TipoEvento(Enum):
    'Identificadores para tipos de eventos'
    Payload = 1
    Timeout = 2
    Frame = 3
    ACK = 4


@attr.s
class Evento:
    'Representa um evento, contendo seu tipo e os dados a ele associados'
    tipo = attr.ib(default=TipoEvento.Timeout)
    msg = attr.ib(factory=bytes)


class Arq(Layer):
    PAYLOAD = TipoEvento.Payload  # vai enviar uma payload
    FRAME = TipoEvento.Frame  # recebeu um frame
    WAIT = TipoEvento.ACK  # esperando um ACK
    TIMEOUT = TipoEvento.Timeout  # evento timeout

    def __init__(self, fd):
        Layer.__init__(self, fd, 100)
        self._seq_RX = b'\x80'
        self._seq_TX = True

        self.evento = Evento(tipo=None, msg=b'0x00')
        #self.quadro = Quadro()
        # self.quadro.set_id_proto(self,IPV4)
        self.Estados = Enum('Estados', 'Idle Wait Backoff B2')
        self.estado = self.Estados.Ocioso

        self.seq_num = 0
        self.restrans_cont = 0
        self.status = True  # bloqueada False, livre True

     # Métodos do Callback

    def handle(self):
        pass

    def handle_timeout(self):
        if (self.status):
            print("ARQ: Timeout!")
            self.handle_fsm(TipoEvento.Timeout)
            pass

    def handle_fsm(self, e: Evento):

        if(self.estado == self.Idle):
            # se recebe dado do canal -> se rx_seq = seq -> (atualiza data e rx) -> envia p app payload -> envia p canal ack rx e faz ack=nack
            if(e.evento.tipo == FRAME):           # recebe dado do canal
                rx_seq = e.msg.get_num_seq
                if(self._seq_RX == e.msg.rx_seq):   # se rx_seq = seq
                    payload_rx = e.msg.get_payload()  # (atualiza data e rx)
                    _seq_RX = ~_seq_RX                # seq = not seq

                    # envia p app payload
                    self.upper.notifica(payload_rx)
                    self.lower.envia(self, self.seq_num)  # envia p canal ack
                    seq_num = ~seq_num                # e faz ack=nack  **************
                    pass
                    # se recebe dado do canal -> se rx_seq != seq -> status não muda -> reenvia ultimo ack p canal
                else:  # se rx_seq != seq
                    self.lower.envia(self, self.seq_num)  # envia p canal ack
                    pass
            else:
                # se recebe dado payload -> envia dado -> Estado vira WAIT
                if(e.evento.tipo == PAYLOAD):
                    self.lower.envia(self, e.msg)
                    self.estado = self.Wait
                    pass
                pass

        if(self.estado == self.Wait):
            # se recebe dado do canal -> se rx_seq = seq -> envia dado para o app -> envia para o canal ACK -> ACK = NACK
            if(e.evento.tipo == FRAME):           # recebe dado do canal
                if(self._seq_RX == e.msg.rx_seq):   # se rx_seq = seq
                    payload_rx = e.msg.get_payload()
                    # envia p app payload
                    self.upper.notifica(payload_rx)
                    self.envia_ack(self, self.seq_num)  # envia p canal ack
                    seq_num = ~seq_num                # e faz ack=nack  **************
                    pass
                # se recebe dado do canal -> se rx_seq =! seq ->status não muda -> reenvia ultimo ack p canal
                else:
                    # reenvia ultimo ack p canal
                    self.envia_ack(self, self.seq_num)
                    pass
            elif(e.evento.tipo == ACK):  # se canal recebe ack,tx
                if(e.evento.msg == self._seq_TX):
                    _seq_TX = ~_seq_TX                  # tx = not tx
                    self.enable_timeout  # Inicia timer (BACKOFF de garantia)
                    self.estado = self.B2
                pass
            elif(e.evento.tipo == TIMEOUT | e.evento.tipo == ACK):  # se Timeout OU canal recebe ack,
                _seq_TX = ~_seq_TX            # not tx->
                self.enable_timeout  # Inicia timer ->
                self.estado = self.estado.Backoff  # entra em backoff
                pass
            else:
                # Pra não deixar nenhuma possibilidade sem tratamento
                print('[FSM WAIT] ERRO 404 -> Estado nao tratado')

        if (self.estado == self.Backoff):
            # se canal recebe dado e not rx -> envia p canal ack e not rx
            if(e.evento.tipo == FRAME):           # recebe dado do canal
                rx_seq = e.msg.get_num_seq
                if(self._seq_RX == ~e.msg.rx_seq):   # se rx_seq = ~seq
                    self.envia_ack(self, self.seq_num)  # envia p canal ack
                    self._seq_RX = ~self._seq_RX
                    pass
                  # se canal recebe dado e rx_seq -> envia para APP o payload -> envia para canal o ACK rx -> rx = not rx
                if(e.evento.tipo == FRAME):           # recebe dado do canal
                    if(self._seq_RX == e.msg.rx_seq):   # se rx_seq = seq
                        payload_rx = e.msg.get_payload()
                        # envia p app payload
                        self.upper.notifica(payload_rx)
                        self.envia_ack(self, self.seq_num)  # envia p canal ack
                        seq_num = ~seq_num                # e faz ack=nack  **************
            # se timer ok -> estado -> idle
            pass
        if (self.estado == self.B2):
            # se canal recebe dado e rx -> envia payload
            if(e.evento.tipo == FRAME):           # recebe dado do canal
                if(self._seq_RX == e.msg.rx_seq):   # se rx_seq = seq
                    payload_rx = e.msg.get_payload()  # envia p app payload
                    self.upper.notifica(payload_rx)
                    pass
            # se canal recebe dado e nrx -> envia nrx
                    self.lower.envia(self, ~self.seq_num)  # envia p canal ack
            # se timer acaba -> estado = ocioso
            if (e.evento.tipo == TIMEOUT):
                self.estado = self.estado.Ocioso
                pass

    # Métodos da FSM

    def envia_ack(self, ack):
        print('ARQ envia: ', ack)
        quadro = Quadro
        self.quadro.set_tipo_ack()  # 0x80
        self.quadro.set_payload(ack)
        envia(self, self.quadro)
        pass

    def envia_dado(self, payload: bytes):
        self.quadro.set_tipo_dados()  # 0x00
        self.quadro.set_payload(payload)
        envia(self, self.quadro)
        seq_tx = not seq_tx
        pass

    def recebe_frame(self, frame: Quadro):
        self.evento.tipo = FRAME
        self.evento.msg = frame
        self.handle_fsm(self.evento)
        pass

    def recebe_payload(self, payload: bytes):
        self.evento.tipo = PAYLOAD
        self.evento.msg = payload
        self.handle_fsm(self.evento)
        pass

    # Métodos do padrão Layer

    def envia(self, frame):  # payload da tun_layer.
        self.lower.envia(frame)
        pass
        #print('ARQ - Recebeu um pyload da tun')
        #e = Evento(tipo=TipoEvento.Payload, msg=frame)
        # if self.status:
        #    self.handle_fsm(e)

        # self.lower.envia(frame)

    def notifica(self, frame):  # frame do enquadramento.
        self.upper.notifica(frame)
        pass

        #e = Evento(tipo=TipoEvento.Payload, msg=frame)
        # if self.status:
        #    self.handle_fsm(e)

        # self.lower.envia(frame)
