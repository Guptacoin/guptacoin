Release Process
====================

* update translations (ping wumpus, Diapolo or tcatm on IRC)
* see https://github.com/bitcoin/bitcoin/blob/master/doc/translation_process.md#syncing-with-transifex

* * *

###update (commit) version in sources


	auroracoin-qt.pro
	contrib/verifysfbinaries/verify.sh
	doc/README*
	share/setup.nsi
	src/clientversion.h (change CLIENT_VERSION_IS_RELEASE to true)

###tag version in git

	git tag -s v(new version, e.g. 2.9.1)

###write release notes. git shortlog helps a lot, for example:

	git shortlog --no-merges v(current version, e.g. 0.7.2)..v(new version, e.g. 0.8.0)

* * *

###setup gitian builds

 	git clone git://github.com/auroracoin/AuroracoinProject.git
    	git clone git://github.com/devrandom/gitian-builder.git
 	
 	cd gitian-builder
    	bin/make-base-vm --suite precise --arch i386
    	bin/make-base-vm --suite precise --arch amd64
    	cd ..

##perform gitian builds

 From a directory containing the auroracoin source, gitian-builder and gitian.sigs
  
	export SIGNER=(your gitian key, ie bluematt, sipa, etc)
	export VERSION=(new version, e.g. 2.9.1)
	pushd ./AuroracoinProject
	git checkout v${VERSION}
	popd
	pushd ./gitian-builder

 Fetch and build inputs: (first time, or when dependency versions change)

	mkdir -p inputs; cd inputs/
	wget 'http://miniupnp.free.fr/files/download.php?file=miniupnpc-1.9.tar.gz' -O miniupnpc-1.9.tar.gz
	wget 'https://www.openssl.org/source/openssl-1.0.1g.tar.gz'
	wget 'http://download.oracle.com/berkeley-db/db-4.8.30.NC.tar.gz'
	wget 'http://zlib.net/zlib-1.2.8.tar.gz'
	wget 'ftp://ftp.simplesystems.org/pub/png/src/history/libpng16/libpng-1.6.8.tar.gz'
	wget 'https://fukuchi.org/works/qrencode/qrencode-3.4.3.tar.bz2'
	wget 'https://downloads.sourceforge.net/project/boost/boost/1.55.0/boost_1_55_0.tar.bz2'
	wget 'https://svn.boost.org/trac/boost/raw-attachment/ticket/7262/boost-mingw.patch' -O \ 
	     boost-mingw-gas-cross-compile-2013-03-03.patch
	wget 'https://download.qt-project.org/official_releases/qt/5.2/5.2.0/single/qt-everywhere-opensource-src-5.2.0.tar.gz'
	wget 'https://protobuf.googlecode.com/files/protobuf-2.5.0.tar.bz2'
	cd ..
	./bin/gbuild ../AuroracoinProject/contrib/gitian-descriptors/boost-linux.yml
	mv build/out/boost-*.zip inputs/
	./bin/gbuild ../AuroracoinProject/contrib/gitian-descriptors/deps-linux.yml
	mv build/out/auroracoin-deps-*.zip inputs/
	./bin/gbuild ../AuroracoinProject/contrib/gitian-descriptors/boost-win.yml
	mv build/out/boost-*.zip inputs/
	./bin/gbuild ../AuroracoinProject/contrib/gitian-descriptors/deps-win.yml
	mv build/out/auroracoin-deps-*.zip inputs/
	./bin/gbuild ../AuroracoinProject/contrib/gitian-descriptors/qt-win.yml
	mv build/out/qt-*.zip inputs/
	./bin/gbuild ../AuroracoinProject/contrib/gitian-descriptors/protobuf-win.yml
	mv build/out/protobuf-*.zip inputs/

 Build auroracoind and auroracoin-qt on Linux32, Linux64, and Win32:
  
	./bin/gbuild --commit auroracoin=v${VERSION} ../AuroracoinProject/contrib/gitian-descriptors/gitian-linux.yml
	./bin/gsign --signer $SIGNER --release ${VERSION} --destination ../gitian.sigs/ ../AuroracoinProject/contrib/gitian-descriptors/gitian-linux.yml
	pushd build/out
	zip -r auroracoin-${VERSION}-linux-gitian.zip *
	mv auroracoin-${VERSION}-linux-gitian.zip ../../../
	popd
	./bin/gbuild --commit auroracoin=v${VERSION} ../AuroracoinProject/contrib/gitian-descriptors/gitian-win.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-win --destination ../gitian.sigs/ ../AuroracoinProject/contrib/gitian-descriptors/gitian-win.yml
	pushd build/out
	zip -r auroracoin-${VERSION}-win-gitian.zip *
	mv auroracoin-${VERSION}-win-gitian.zip ../../../
	popd
	popd

  Build output expected:

  1. linux 32-bit and 64-bit binaries + source (auroracoin-${VERSION}-linux-gitian.zip)
  2. windows 32-bit and 64-bit binaries + installer + source (auroracoin-${VERSION}-win-gitian.zip)
  3. Gitian signatures (in gitian.sigs/${VERSION}[-win]/(your gitian key)/

repackage gitian builds for release as stand-alone zip/tar/installer exe

**Linux .tar.gz:**

	unzip auroracoin-${VERSION}-linux-gitian.zip -d auroracoin-${VERSION}-linux
	tar czvf auroracoin-${VERSION}-linux.tar.gz auroracoin-${VERSION}-linux
	rm -rf auroracoin-${VERSION}-linux

**Windows .zip and setup.exe:**

	unzip auroracoin-${VERSION}-win-gitian.zip -d auroracoin-${VERSION}-win
	mv auroracoin-${VERSION}-win/auroracoin-*-setup.exe .
	zip -r auroracoin-${VERSION}-win.zip auroracoin-${VERSION}-win
	rm -rf auroracoin-${VERSION}-win

**Perform Mac build:**

  OSX binaries are created by Gavin Andresen on a 64-bit, OSX 10.6 machine.

	./autogen.sh
        SDK=$(xcode-select --print-path)/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.6.sdk
        CXXFLAGS="-mmacosx-version-min=10.6 -isysroot $SDK" ./configure --enable-upnp-default
	make
	export QTDIR=/opt/local/share/qt4  # needed to find translations/qt_*.qm files
	T=$(contrib/qt_translations.py $QTDIR/translations src/qt/locale)
        export CODESIGNARGS='--keychain ...path_to_keychain --sign "Developer ID Application: BITCOIN FOUNDATION, INC., THE"'
	python2.7 contrib/macdeploy/macdeployqtplus Auroracoin-Qt.app -sign -add-qt-tr $T -dmg -fancy contrib/macdeploy/fancy.plist

 Build output expected: Auroracoin-Qt.dmg

###Next steps:

* Code-sign Windows -setup.exe (in a Windows virtual machine using signtool)
 Note: only Gavin has the code-signing keys currently.

* upload builds to SourceForge

* create SHA256SUMS for builds, and PGP-sign it

* update bitcoin.org version
  make sure all OS download links go to the right versions
  
* update download sizes on bitcoin.org/_templates/download.html

* update forum version

* update wiki download links

* update wiki changelog: [https://en.bitcoin.it/wiki/Changelog](https://en.bitcoin.it/wiki/Changelog)

Commit your signature to gitian.sigs:

	pushd gitian.sigs
	git add ${VERSION}/${SIGNER}
	git add ${VERSION}-win/${SIGNER}
	git commit -a
	git push  # Assuming you can push to the gitian.sigs tree
	popd

-------------------------------------------------------------------------

### After 3 or more people have gitian-built, repackage gitian-signed zips:

From a directory containing auroracoin source, gitian.sigs and gitian zips

	export VERSION=(new version, e.g. 0.8.0)
	mkdir auroracoin-${VERSION}-linux-gitian
	pushd auroracoin-${VERSION}-linux-gitian
	unzip ../auroracoin-${VERSION}-linux-gitian.zip
	mkdir gitian
	cp ../auroracoin/contrib/gitian-downloader/*.pgp ./gitian/
	for signer in $(ls ../gitian.sigs/${VERSION}/); do
	 cp ../gitian.sigs/${VERSION}/${signer}/auroracoin-build.assert ./gitian/${signer}-build.assert
	 cp ../gitian.sigs/${VERSION}/${signer}/auroracoin-build.assert.sig ./gitian/${signer}-build.assert.sig
	done
	zip -r auroracoin-${VERSION}-linux-gitian.zip *
	cp auroracoin-${VERSION}-linux-gitian.zip ../
	popd
	mkdir auroracoin-${VERSION}-win-gitian
	pushd auroracoin-${VERSION}-win-gitian
	unzip ../auroracoin-${VERSION}-win-gitian.zip
	mkdir gitian
	cp ../auroracoin/contrib/gitian-downloader/*.pgp ./gitian/
	for signer in $(ls ../gitian.sigs/${VERSION}-win/); do
	 cp ../gitian.sigs/${VERSION}-win/${signer}/auroracoin-build.assert ./gitian/${signer}-build.assert
	 cp ../gitian.sigs/${VERSION}-win/${signer}/auroracoin-build.assert.sig ./gitian/${signer}-build.assert.sig
	done
	zip -r auroracoin-${VERSION}-win-gitian.zip *
	cp auroracoin-${VERSION}-win-gitian.zip ../
	popd

- Upload gitian zips to SourceForge

- Announce the release:

  - Add the release to bitcoin.org: https://github.com/auroracoin/bitcoin.org/tree/master/_releases

  - Release sticky on auroracointalk: https://bitcointalk.org/index.php?board=1.0

  - Auroracoin-development mailing list

  - Optionally reddit /r/Auroracoin, ...

- Celebrate 
