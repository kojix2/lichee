

lichee.jar:
	rm -rf build/; \
	mkdir build/; \
	javac -g -classpath $$(ls -1 lib/*.jar | tr "\n" ":") -sourcepath src/ -d build/ src/lineage/LineageEngine.java; \
	cp -r lib/org MANIFEST.MF lib/*.jar build/; \
	cd build/; \
	jar -cfm ../release/lichee.jar MANIFEST.MF lineage/*.class util/*.class *.jar org/eclipse/jdt/internal/jarinjarloader/*.class; \
	cd -;