#################################################################################
#										#
# Makefile for libtpm/lib							#
#										#
# $Id: Makefile.am 4073 2010-04-30 14:44:14Z kgoldman $				#
#										#
# (c) Copyright IBM Corporation 2006, 2010.					#
# 										#
# All rights reserved.								#
# 										#
# Redistribution and use in source and binary forms, with or without		#
# modification, are permitted provided that the following conditions are	#
# met:										#
# 										#
# Redistributions of source code must retain the above copyright notice,	#
# this list of conditions and the following disclaimer.				#
# 										#
# Redistributions in binary form must reproduce the above copyright		#
# notice, this list of conditions and the following disclaimer in the		#
# documentation and/or other materials provided with the distribution.		#
# 										#
# Neither the names of the IBM Corporation nor the names of its			#
# contributors may be used to endorse or promote products derived from		#
# this software without specific prior written permission.			#
# 										#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS		#
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT		#
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR		#
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT		#
# HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,	#
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT		#
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,		#
# DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY		#
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT		#
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE		#
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.		#
#										#
#################################################################################

AM_CFLAGS = -I/usr/linux/include -I../include -Wall -ggdb -Wuninitialized -Wmissing-declarations -Wmissing-prototypes -Wnested-externs -W -DTPM_POSIX -DTPM_V12 -DTPM_NV_DISK -DTPM_AES -DTPM_USE_TAG_IN_STRUCTURE=1 -DTPM_USE_CHARDEV
noinst_LTLIBRARIES = libtpm.la


noinst_HEADERS = tpmfunc.h \
	tpm.h \
	tpmkeys.h \
	tpm_constants.h \
	tpm_error.h \
	tpm_structures.h \
	tpmutil.h \
	tpm_types.h \
	tpm_lowlevel.h \
	oiaposap.h

libtpm_la_SOURCES = auditing.c bind.c chgauth.c context.c \
	counter.c daa.c debug.c delegation.c dir.c \
	eviction.c hmac.c identity.c keys.c keyswap.c \
	maintenance.c management.c migrate.c miscfunc.c nv.c \
	oiaposap.c optin.c owner.c ownertpmdiag.c \
	pcrs.c raw.c rng.c seal.c serialize.c session.c \
	sha.c signature.c startup.c testing.c \
	ticks.c tpmutil.c tpmutil_sock.c tpmutil_tty.c tpmutil_unixio.c \
	transport.c

