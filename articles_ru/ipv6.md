 IPv6 and Friends
Как уже понятно из топика сегодня речь пойдёт о IPv6, а точнее о механизме ND он же Neighbor Discovery. Для решения всех вопросов связанных с определением MAC адресов в IPv4 используется специальный протокол ARP (address resolution protocol). К IPv6 ARP никакого отношения не имеет, в новом протоколе используется новый механизм для определения адресов, который реализован на обмене ICMPv6 пакетами. В момент когда один компьютер вдруг решил что хочет отправить пакет другому компьютеру, то посылает мультикаст запрос Neighbor Solicitation Message: "а кто такой с адресом блаблабла", в ответ должен получить Neighbor Advertisement Message: "это я! это я!". И таким образом узнать MAC адресс другого компьютера.

# Allow ND Solicitation
ip6tables -A FORWARD -p icmpv6 --icmpv6-type neighbor-solicitation -j ACCEPT
ip6tables -A FORWARD -p icmpv6 --icmpv6-type neighbor-advertisement -j ACCEPT

Кроме этого не хитрого механизма, предусмотрен ещё другой механизм -- автоматическое определение роуетра. Соотвецтвенно нужно не забывать учитывать или не учитывать это в фаерволе:

ip6tables -A FORWARD -p icmpv6 --icmpv6-type router-solicitation -j DROP
ip6tables -A FORWARD -p icmpv6 --icmpv6-type router-advertisement -j DROP