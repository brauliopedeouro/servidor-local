require 'socket'
require 'rdbi-driver-sqlite3'

socket = UDPSocket.new
socket.bind("", 2100)
puts "Central Server Online!!!"
dbh = RDBI.connect(:SQLite3, :database => "reg_dominios.db")
aspas = '"'
reply = nil
loop {
	puts "Connected"
	armazenaSolicitacao, sender = socket.recvfrom(1024)
	puts armazenaSolicitacao
	solicitacao = armazenaSolicitacao.split
	c_ip = sender[3]
	c_port = sender[1]
	if solicitacao[0] == "REG"
		if solicitacao[1] != nil && solicitacao[2] != nil
			begin
				puts "##### RECEBENDO SOLICITACAO DE REGISTRO DE DOMINIO"
				dbh.execute("insert into REGISTROS (DOMINIO, IP) values ( #{aspas}#{solicitacao[1]}#{aspas}, #{aspas}#{solicitacao[2]}#{aspas})")
				puts "##### REGOK ##### REGISTRO DE DOMINIO REALIZADO COM SUCESSO"
				socket.send "REGOK", 0 , c_ip, c_port
			rescue
				puts "##### REGFALHA ##### O DOMINIO JA SE ENCONTRA REGISTRADO"
				socket.send "REGFALHA", 0, c_ip, c_port
			end
		else
			puts "##### FALHA INESPERADA #####"
			socket.send "FALHA", 0, c_ip, c_port
		end
	elsif solicitacao[0] == "IP"
		if solicitacao[1] != nil
			puts "##### RECEBENDO SOLICITACAO DE IP #####"
			rs = dbh.execute("select IP from REGISTROS where DOMINIO = #{aspas}#{solicitacao[1]}#{aspas}")
			rs.fetch(:all).each do |row|
				reply = row
			end
			if reply != nil
				puts "##### IPOK ##### ENVIANDO IP"
				socket.send "IPOK #{reply}", 0, c_ip, c_port
			elsif reply == nil
				puts "##### IPFALHA ##### ENDERECO IP NAO ENCONTRADO"
				socket.send "IPFALHA", 0, c_ip, c_port
			end
		else
			puts "##### FALHA INESPERADA #####"
			socket.send "FALHA", 0, c_ip, c_port
		end
	else
		puts "##### FALHA INESPERADA #####"
		socket.send "FALHA", 0, c_ip, c_port	
	end
}
socket.close
